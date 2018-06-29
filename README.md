# DEx.top - Instant Trading on Chain

Exchange: https://dex.top

## Overview

DEx.top achieves instant trading experience with its two-ledger architecture:
an off-chain ledger responsible for providing instant trading experience and a
smart-contract based on-chain ledger to ensure the traders' assets are safe.

The two ledgers are closely synchronized, where all trading activities are first
registered on the off-chain ledger and later get exactly replayed (executed in
the same way and the same order) and confirmed on the on-chain ledger. The
on-chain ledger can be interpreted as a delayed version of the off-chain ledger,
which will eventually catch up.

See the [White Paper](./whitepaper/DEx-Whitepaper-Short-Version.pdf) for details.

## Smart Contract
It is deployed at [0x7600977](https://etherscan.io/address/0x7600977eb9effa627d6bd0da2e5be35e11566341) on Ethereum.
See the [Spec](./smart-contract/dextop-spec.md), the source code of [dextop.sol](./smart-contract/dextop.sol), and the [Gas Cost Chart](./smart-contract/dextop-gas-cost.md) for details.

Here is an article introducing the gas cost saving methods used in DEx.top
[link](https://medium.com/coinmonks/techniques-to-cut-gas-costs-for-your-dapps-7e8628c56fc9).

## License
DEx.top is under the Apache 2.0 license. See the [LICENSE](./LICENSE) file for details.
