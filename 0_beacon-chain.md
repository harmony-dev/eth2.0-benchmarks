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
Just a couple of things:
- `compute_committee` uses [optimized shuffling](https://github.com/protolambda/eth2-shuffle/blob/master/shuffle.go#L159)
- genesis and next to it epochs are always skipped as they are not completely processed

### Epoch processing complexity
It obvious that epoch processing has at _least_ `O(n)` complexity where `n` is validator registry size.
Let's take a closer look to it by comparing a number of method call times made during epoch processing 
with various registry sizes.

#### v0.6.1
Number of function calls counted during epoch processing with various registry sizes.

| Method                                      | 1k validators | 10k validators | 100k validators |
|---------------------------------------------|--------------:|---------------:|----------------:|
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
<strong>Note:</strong> `get_attesting_indices` is called _N*M_ times where _M_ is number of shards per epoch, if registry size is _1m_ then it will at least be called `1,024,000,000` times in the best case (the case when each shard has a single aggregate attestation per slot).

#### Complexity mitigation
After taking a look at call dependencies following method results have been cached:
- get_crosslink_committee
- get_attesting_indices
- get_active_validator_indices
- get_total_active_balance

These caches drastically reduces an impact of registry size growth on epoch processing time.

Take a look at a table below which compares cumulative times of method calls of epoch processing,
measured with 10,000 registry size:

| Routine                      | Cached, ms | Not cached, ms |
|------------------------------|-----------:|---------------:|
| epoch processing             | 2,910      | 29,447         |
| get_attesting_indices        | 1,778      | 10,551         |
| get_base_reward              | 253        | 18,047         |
| get_total_active_balance     | 16         | 17,980         |
| get_total_balance            | 7          | 17,937         |
| get_active_validator_indices | 2          | 17             |


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
