# Beacon chain benchmarks

This page contains description and analysis of data measured on execution of [Beacin chain spec](https://github.com/ethereum/eth2.0-specs/blob/dev/specs/core/0_beacon-chain.md).

## Tooling
Data in following sections has been collected with usage of [benchmaker](https://github.com/harmony-dev/beacon-chain-java/wiki/Benchmaker), a helper tool that benchmarking beacon chain spec.
Benchmaker uses simulation to collect data, i.e. attestations and blocks are produced by validators in a runtime.
Number of validators may be different from one session to another and is set via CLI option.
There are other options used to turn off BLS, caches, etc.

## Consensus
This section covers slot, block and epoch processing in a part which is related to attestations, crosslinks, rewards, penalties and finality. 
In one word everything that is somehow related to the consensus part of the spec. 

Operations other than attestations are not covered by this section.
Fork and fork choice rule is not benchmarked either and as there is only one chain blocks carry only one aggregate attestation.

### Setup
General setup note worth mentioning is that all benchmarks in this section uses 
[optimized shuffling](https://github.com/protolambda/eth2-shuffle/blob/master/shuffle.go#L159) 
algorithm instead of the one described in the spec. 
As first two epochs are partly used by epoch processing benchmaker uses first two epochs to warm up caches and hasher and starts measurements only from 3rd epoch.
Other noticeable options will be listed within each part.

### Epoch processing complexity
It obvious that epoch processing has at _least_ `O(n)` complexity where `n` is validator registry size.
Let's take a closer look to it by comparing a number of method call times made during epoch processing 
with various registry sizes.

#### v0.6.1
Number of function calls counted during epoch processing with various registry sizes.

| Method                                      | 1k validators | 10k validators |
|---------------------------------------------|--------------:|---------------:|
| **get_crosslink_committee**                 | 64,950        | 640,950        |
| **get_attesting_indices**                   | 64,758        | 640,758        |
| **get_active_validator_indices**            | 7,034         | 61,034         |
| **get_total_balance**                       | 6,393         | 60,393         |
| **get_total_active_balance**                | 6,003         | 60,003         |
| **get_base_reward**                         | 6,000         | 60,000         |
| get_epoch_committee_count                   | 964           | 964            |
| get_shard_delta                             | 833           | 833            |
| get_epoch_start_shard                       | 320           | 320            |
| hash_tree_root                              | 311           | 311            |
| get_unslashed_attesting_indices             | 196           | 196            |
| get_winning_crosslink_and_attesting_indices | 192           | 192            |
| get_attestation_slot                        | 64            | 64             |
| generate_seed                               | 64            | 64             |
| get_attesting_balance                       | 5             | 5              |
| get_churn_limit                             | 1             | 1              |

<sup>*</sup> Methods referring to `O(n)` complexity are **highlighted**.

#### Complexity mitigation
After taking a look at call dependencies graph following method calls have been cached:
- get_crosslink_committee
- get_attesting_indices
- get_active_validator_indices
- get_total_active_balance

These caches almost nullify an impact of registry size growth on epoch processing time.

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
