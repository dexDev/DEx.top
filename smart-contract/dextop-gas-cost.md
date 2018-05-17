## Trading Associated Gas Costs

|    Operation                | Gas (Avg.)| ETH (3Gwei)   | ETH (5Gwei)   | ETH (10Gwei)  |  Saving |
|:----------------------------|----------:|:-------------:|:-------------:|:-------------:|:-----------:|
|MatchOrders_BothNew 1x       |   136159  |   0.00040848  |   0.00068079  |   0.00136159  | Base |
|MatchOrders_BothNew 2x       |   121840  |   0.00036552  |   0.00060920  |   0.00121840  | **11%** |
|MatchOrders_BothNew 4x       |   114880  |   0.00034464  |   0.00057440  |   0.00114880  | **16%** |
|MatchOrders_BothNew 8x       |   111972  |   0.00033592  |   0.00055986  |   0.00111972  | **18%**
|MatchOrders_ExistingAndNew 1x|   110885  |   0.00033266  |   0.00055442  |   0.00110885  | Base |
|MatchOrders_ExistingAndNew 2x|    96362  |   0.00028909  |   0.00048181  |   0.00096362  | **13%** |
|MatchOrders_ExistingAndNew 4x|    89300  |   0.00026790  |   0.00044650  |   0.00089300  | **19%** |
|MatchOrders_ExistingAndNew 8x|    85821  |   0.00025746  |   0.00042910  |   0.00085821  | **23%** |
|MatchOrders_BothExisting 1x  |    85606  |   0.00025682  |   0.00042803  |   0.00085606  | Base |
|MatchOrders_BothExisting 2x  |    71146  |   0.00021344  |   0.00035573  |   0.00071146  | **17%** |
|MatchOrders_BothExisting 4x  |    63918  |   0.00019175  |   0.00031959  |   0.00063918  | **25%** |
|MatchOrders_BothExisting 8x  |    60305  |   0.00018091  |   0.00030153  |   0.00060305  | **30%** |
|HardCancelOrder              |    37813  |   0.00011344  |   0.00018907  |   0.00037813  | N/A |

**Note**

1. Terms `MatchOrders_BothNew`, `MatchOrders_ExistingAndNew`, and `MatchOrders_BothExisting` refer to the scenarios
where a) both orders are new, b) one of them has been partially filled before and c) both have been partially filled, respectively.

2. The **batch execution** approach of DEx.top can greatly reduced the amortized gas cost. In the `operation` column *N*x means that the batch size is *N*. As showed in the above table batch execution can reduced the average gas cost of executing a pair of matched orders by 18% ~ 30% when *N* is 8.

3. `HardCancelOrder` is the operation of marking an order as cancelled (can no longer be matched) in the smart contract. Hard cancellation is enforced by the smart contract.
