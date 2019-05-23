# Beacon chain benchmarks

This page contains description and analysis of data measured from execution of [Beacin chain spec](https://github.com/ethereum/eth2.0-specs/blob/dev/specs/core/0_beacon-chain.md).

## Tooling
Data in following sections has been collected with usage of [benchmaker](https://github.com/harmony-dev/beacon-chain-java/wiki/Benchmaker), a helper tool that benchmarking beacon chain spec.
Benchmaker uses simulation to collect data, i.e. attestations and blocks are produced by validators in a runtime.
Number of validators may be different from one session to another and is set via CLI option.
There are other options used to turn off BLS, caches, etc.

## Consensus
This section covers slot, block and epoch processing in a part which is related to attestations, crosslinks, rewards, penalties and finality. 
In other words everything that is related to consensus part of the spec. 

Operations other than attestations are not used in this section.
Fork choice rule is not measured either and since runtime generates a single canonical chain number of aggregate attestations per block equals to a number of committees per slot.

### Setup
A couple of general things:
- `compute_committee` uses [optimized shuffling](https://github.com/protolambda/eth2-shuffle/blob/master/shuffle.go#L159)
- genesis and next to it epochs are always skipped as they are not completely processed

Laptop configuration:
- MacBook Pro
- 2.3GHz quad-core Intel Core i7
- 16GB of 1600MHz DDR3L

### Epoch processing complexity
It obvious that epoch processing has at _least_ `O(n)` complexity where `n` is validator registry size.
Let's take a closer look to it by comparing a number of method call times made during epoch processing 
with various registry sizes.

#### v0.6.1
Number of function calls counted during epoch processing with various registry sizes.

| Routine                                     | 1k validators | 10k validators | 100k validators |
|:--------------------------------------------|--------------:|---------------:|----------------:|
| **get_attesting_indices**                   | 64,758        | 640,758        | 76,808,644      |
| **get_crosslink_committee**                 | 64,950        | 640,950        | n/a             |
| **get_active_validator_indices**            | 7,034         | 61,034         | n/a             | 
| **get_total_balance**                       | 6,393         | 60,393         | n/a             | 
| **get_total_active_balance**                | 6,003         | 60,003         | 600,003         | 
| **get_base_reward**                         | 6,000         | 60,000         | 600,000         | 
| get_epoch_committee_count                   | 964           | 964            | 22,864          | 
| get_shard_delta                             | 833           | 833            | 18,313          | 
| get_epoch_start_shard                       | 320           | 320            | 6,852           | 
| hash_tree_root                              | 311           | 311            | 3,721           | 
| generate_seed                               | 64            | 64             | 3,780           | 
| get_winning_crosslink_and_attesting_indices | 192           | 192            | 2,304           | 
| get_unslashed_attesting_indices             | 196           | 196            | 1,801           | 
| get_attestation_slot                        | 64            | 64             | 768             | 
| get_attesting_balance                       | 5             | 5              | 5               | 
| get_churn_limit                             | 1             | 1              | 1               | 

<sup>*</sup> Methods referring to `O(n)` complexity are **highlighted**.

<strong>Note:</strong> `get_attesting_indices` has _N*M_ calls where _M_ is number of shards crosslinked per epoch, if registry size is _1m_ then this function will be called `1,024,000,000` times in the best case (the case when each shard has a single aggregate attestation per slot).

#### Complexity mitigation
After taking a look at call dependencies following method results have been cached:
- get_crosslink_committee
- get_attesting_indices
- get_active_validator_indices
- get_total_active_balance

These caches drastically reduces an impact of registry size growth on epoch processing time.

Take a look at a table below which compares cumulative times of method calls of epoch processing,
measured with 10,000 registry size:

| Routine                      | Cached, s  | Not cached, s  |
|:-----------------------------|-----------:|---------------:|
| epoch processing             | 2.910      | 29.447         |
| get_attesting_indices        | 1.778      | 10.551         |
| get_base_reward              | 0.253      | 18.047         |
| get_total_active_balance     | 0.016      | 17.980         |
| get_total_balance            | 0.007      | 17.937         |
| get_active_validator_indices | 0.002      | 0.017          |


Cumulative time of epoch processing routines with 100,000 registry size (768 committees per epoch):

| Routine                                     |      Count | Time, s  |
|:--------------------------------------------|-----------:|---------:|
| epoch processing                            | 1          | 329.063  |
| get_attesting_indices                       | 76,808,644 | 229.031  |
| get_crosslink_committee                     | 5,256      | 36.195   |
| get_unslashed_attesting_indices             | 1,801      | 11.516   |
| get_attesting_balance                       | 5          | 7.107    |
| get_base_reward                             | 60,0000    | 1.127    |
| get_winning_crosslink_and_attesting_indices | 2,304      | 0.505    |

<sup>*</sup> caches are enabled.

#### Conclusion
Further optimization is required to handle realistic registry sizes like 1,000,000 validators, it highly likely involves optimization of some algorithms written in the spec.

#### Used commands
1k validators:
```bash
./benchmaker --no-bls --epochs 1 --registry-size 1000
./benchmaker --no-bls --epochs 1 --registry-size 1000 --no-cache
```
10k validators:
```bash
./benchmaker --no-bls --epochs 1 --registry-size 10000
./benchmaker --no-bls --epochs 1 --registry-size 10000 --no-cache
```
100k validators:
```bash
./benchmaker --no-bls --epochs 1 --registry-size 100000
```

### Block processing
Cumulative time of block processing routines with 100,000 registry size (12 committees per slot):

| Routine              | Count | Time, s  |
|:---------------------|------:|---------:|
| block processing     | 1     | 7.130    |
| process_attestation  | 12    | 6.974    |
| process_block_header | 1     | 0.078    |
| process_randao       | 1     | 0.077    |
| process_eth1_data    | 1     | 0.000    |

<sup>*</sup> BLS is enabled.

#### BLS
_99%_ of block processing time is taken by BLS:

| Routine               | Count | Time, s  |
|:----------------------|------:|---------:|
| bls_aggregate_pubkeys | 24    | 5.487    |
| bls_verify_multiple   | 12    | 1.418    |
| bls_verify            | 2     | 0.155    |

Two `bls_verify` calls correspond to proposer signature and randao reveal verifications, the others are used by `verify_indexed_attestation` as a part of `process_attestation` routine.

Noticable fact is that public keys aggregation is _2 times_ slower than multiple verification which must not be the case. This is probably caused by `BLSPubkey` verification which happens for each pubkey passed to `bls_aggregate_pubkeys` and causes about _260_ extra EC muls per each processed block.

#### Used commands
```bash
./benchmaker --epochs 1 --registry-size 100000
```
