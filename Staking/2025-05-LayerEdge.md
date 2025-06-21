
# Summary.

LayerEdge is a Bitcoin backed Internet using trust minimized verification & proof aggregation for all layers & apps. This contest focuses on staking contract that reward users as per FCFS model where earlier stakers receive higher rewards.

### Findings.

### [H-1](https://github.com/sherlock-audit/2025-05-layeredge-judging/issues/83) Incorrect tier tracking when tier 3 staker exits in a certain case.

Whenever a `staker` exits and the old T2 and new T2 counts are the same, than we'd end up above. However this code does not properly work when a T3 staker exits from the tree and completely breaks the tier tracking.

```solidity
            else if (isRemoval) {
                _findAndRecordTierChange(new_t1 + new_t2, n);
            }
```

In a scenario below(`reporter is referring to his own test file`), there are `7` T1 & T2 and `8` T3 stakers (15 total) stakers initially. T3 staker exits, so now there must be `2` T1 stakers, `4` T2 stakers and `8` T3 stakers where the last T2 staker is demoted. However, in the snippet shown above, we write to `new_t1 + new_t2` rank which is `6 (2 + 4)`. However, as a `T3` staker exited, then the `7th rank` is the actual last `T2` staker, not the `6th` we are writing to.

## What went wrong and how to fix it next time.

* Didnt review this part of the codebase.
* Didnt look enough to see whether the invariant may break.
  
## Take aways.

* When the docs give you a clear invariant, try everything in my power to break it i.e `using more users to carry out specific actions that may perharps break the invariant`.

## Rewards -> `$778`

### [H-2](https://github.com/sherlock-audit/2025-05-layeredge-judging/issues/74). Incorrect tier updates when adding users.

When new `stakers` stake into the pool, certain staking position may not be automatically promoted to the next tier. As a result, affected position will be applied lower APY than expected. This will cause reward loss from that user.

```solidity
901:             // Handle case where Tier 2 count stays the same
902:             else if (isRemoval) {
903:                 _findAndRecordTierChange(new_t1 + new_t2, n);
904:             } else if (!isRemoval) {
905:@>               _findAndRecordTierChange(old_t1 + old_t2, n);
906:             }
```

## What went wrong and how to fix it next time.

* Didnt review this part of the codebase.
* Didnt look enough to see whether the invariant may break.
  
## Take aways.

* When the docs give you a clear invariant, try everything in my power to break it i.e `using more users to carry out specific actions that may perharps break the invariant`.

## Rewards -> `$167`

### [M-1](https://github.com/sherlock-audit/2025-05-layeredge-judging/issues/150). Unbounded stake/unstake Operations leading to contract Denial Of service.

The contract allows `users` to create an unlimited number of stake/unstake requests. Processing these requests consumes gas linearly with the queue length, eventually exceeding Ethereum’s block gas limit (~30M gas) and rendering the contract unusable. A `user` can stake above the min stake to be added as a tier 1 and then flood the protocol with span 1 wei stakes to make the contract unusable. Or even worse, an `attacker` determined to take down the protocol can use this. Although it cost to execute this attack its still a critical flaw.

## What went wrong and how to fix it next time.

* Didnt review this part of the codebase.


## Take aways.

* When you see a loop: check to see if its unbounded and also check to see if the user can influence that loop to iterate over something huge. Like huge sets of data in an array.

## Rewards -> `$7`