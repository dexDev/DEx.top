# DEx Smart Contract Spec

Bits format in this doc: The rightmost bit is the LSB, the leftmost bit is the MSB. For example, a byte <1001>(4) <0011>(4) has value 147.

# Market States

## State that can not be changed:

- admin: address

## States that can be changed directly by an external function:

- marketStatus: uint8
  - 0: Active
  - 1: Suspended
    - A suspended market temporarily stops accepting deposits and executing operations. Withdrawing is Ok.
  - 2: Closed
    - A closed status cannot be changed even by the admin. Only withdrawing, setWithdrawAddr, transferFee and operation ConfirmDeposit and InitiateWithdraw are allowed in a closed market.

- tokens: `map[tokenCode(16)] -> TokenInfo`
  - tokenCode is a uint16
  - TokenInfo is a struct
  ```
  struct TokenInfo {
      bytes[4] symbol // e.g. "ETH ", "ADX " (there is a trailing whitespace 0x20).
      Address tokenAddr // the address of the ERC20 token contract.
      uint64 scaleFactor // <original amount> = <Dex amountE8> x scaleFactor / 1e8
      uint minDeposit // minimum deposit (original token amount) allowed per token
  }
  ```

  - **Note**

    1. Once a token added, its info cannot be changed.
    2. The token code of ETH is 0 and the scale factor is 1e18.
    3. Token code [0, 99] are reserved for cash tokens. [100, 999] are reserved for tokens on Ethereum block chain.
    4. For an ERC20 token, the scaleFactor of the token must be set as
       scaleFactor = 10 ** decimal. For example, demical=3 => scaleFactor=1000, decimal=0 => scaleFactor=1.

## States that can only be changed via executing the operation sequence:

- makerFeeRateE4: uint16
- takerFeeRateE4: uint16
- lastDepositIndex: uint64
  - Initial value is 0 (i.e. the first deposit has index 1)
- exeStatus: struct ExeStatus

  ```
  struct ExeStatus {
      uint64 logicTimeSec
      uint64 lastOperationIndex
  }
  ```

  - logicTimeSec: uint64
    - This is the time (seconds since the epoch) for checking order expiration.
    - Expires can directly check against this value for validation purpose.
  - lastOperationIndex: uint64
    - Updated upon operation sequence completion.
    - Initial value is 0 (i.e. the first operation has index 1)

# Account States (Trader Related States)

- traders: `map[traderAddr] -> TraderInfo(256)`
  - traderAddr is an address
  - TraderInfo is a struct
  ```
  struct TraderInfo {
      address withdrawAddr
      uint8   feeRebatePercent  // between 0~100
  }
  ```

- accounts: `map[tokenAccountKey(176)] -> TokenAccount(256)`
  - tokenAccountKey is a uint176
    - `<tokenCode>(16) <traderAddr>(160)`
  - TokenAccount is a struct
  ```
  struct TokenAccount {
      uint64 balanceE8  // available amount for trading
      uint64 pendingWithdrawE8
  }
  ```

  - **Note**

    1. The total fee collected in a token is kept as the pendingWithdrawE8 of that token of traderAddr 0, i.e. `accounts[tokenCode << 160].pendingWithdrawE8`
    2. We use address 0 but not a real address to avoid handling the case where the fee address is one of the trader address. Besides, we do not have to set the fee address in advance.

- orders: `map[orderKey(224)] -> Order`
  - orderKey is a uint224
    - `<orderNonce>(64) <traderAddr>(160)`
  - Order is a struct
  ```
  struct Order {
      uint32 pairId  // <cashId>(16) <stockId>(16)
      uint8  action
      uint8  ioc
      uint64 priceE8
      uint64 amountE8
      uint64 expireTimeSec
  }
  ```

  - **Note**

    1. action: 0 means BUY, 1 means SELL.
    2. ioc: 0 means not IOC (Immediate-Or-Cancel), 1 means IOC.
       If an order has been hard cancelled, update its amountE8 to 0.

- deposits: `map[depositIndex(64)] -> Deposit`
  - depositIndex is a uint64
  - Deposit is a struct
  ```
  struct Deposit {
      address traderAddr
      uint16  tokenCode
      uint64  pendingAmountE8  // the amount to be confirmed for trading
  }
  ```

# Operation Sequence

An operation sequence is represented as an array of uint256 with the following format:

`<OpSeq>: <header>(256) {[operation],[operation],...}`

Format of header:

`<header>: <newLogicTimeSec>(64) <beginIndex>(64)`

External function:

`function exeSequence(uint header, uint[] body)`

## Operations

Each operation can be one of the following. As a convention, the lowest 16 bits of the first uint256 of an operation is always the opcode.

- ConfirmDeposit
  - move a pending deposit to balance
  - opcode: 0xDE 0x01
  - body length (in uint256): 1
  - `<depositIndex>(64) <opcode>(16)`

- InitiateWithdraw
  - move fund from balance to pending withdraw
  - opcode: 0xDE 0x02
  - length (in uint256): 1
  - `<amountE8>(64) <tokenCode>(16) <traderAddr>(160) <opcode>(16)`

  - **Note**
    - No signature required. The admin keeps the right to clear a trader account by sending the funds of the trader back to the trader's account.

- MatchOrders
  - execute a pair of matching orders.
  - opcode: 0xDE 0x03
  - body length (in uint256): 2 ~ 8
  - `<nonce1>(64) <traderAddr1>(160) <v1>(8) <opcode>(16)`
    - The following 3 uint256s exists if and only if v1 is NOT 0. (v1 = 0 means that this order is already in the storage.)
    ```
    <expireTimeSec1>(64) <amount1E8>(64) <price1E8>(64) <ioc1>(8) <action1>(8) <pairId1>(32)
    <r1>(256)
    <s1>(256)
    ```

  - `<nonce2>(64) <traderAddr2>(160) <v2>(8)`
    - The following 3 uint256s exists if and only if v2 is NOT 0. (v2 = 0 means that this order is already in the storage.)
    ```
    <expireTimeSec2>(64) <amount2E8>(64) <price2E8>(64) <ioc2>(8) <action2>(8) <pairId2>(32)
    <r2>(256)
    <s2>(256)
    ```

  - **Note**

    1. The first order is the maker, the second order is the taker.
    2. When v1/v2 is not zero, it is either 27 or 28.
    3. Format of `<pairId>: <cashId>(16) <stockId>(16)`
    4. stockId is the lower u16, cashId is the higher u16
    5. AmountE8 is referring to stock amount

  ##### Signing Scheme 1 (Friendly to API usage)

    - The bytes to be hashed (using keccak256) for signing are the concatenation of the following (uints are in big-endian order):
      - Prefix "\x19Ethereum Signed Message:\n70".
      - String "DEx2 Order: " (Note the trailing whitespace)
      - The market address.
        - This is for replay attack protection when we deploy a new market.
      - `<nonce>(64)`
      - `<expireTimeSec>(64) <amountE8>(64) <priceE8>(64) <ioc>(8) <action>(8) <pairId>(32)`

  ##### Signing Scheme 2 (Friendly to UI supporting eth_signTypedData)

    - The fields to be sent to eth_signTypedData:

      ```
      string title = "DEx2 Order"
      address market_address = <market address>
      uint64 nonce
      uint64 expire_time_sec
      uint64 amount_e8
      uint64 price_e8
      uint8 immediate_or_cancel
      uint8 action
      uint16 cash_token_symbol = `<high 16 bits of pairId>`
      uint16 stock_token_symbol = `<low 16 bits of pairId>`
      ```

    - The bytes to be hashed (using keccak256) for signing are the concatenation of the following:

      `keccak256(<TYPES_NAMES_HASH>, <DATA_HASH>)`

      - <TYPES_NAMES_HASH> is `u256 0x8382001f45a579e41d75c5535cff758120963de17b4af5e92d5191b9a5f69f1f`
        - It is the result of:
          ```
          keccak256(
          "string title", "address market_address", "uint64 nonce",
          "uint64 expire_time_sec", "uint64 amount_e8", "uint64 price_e8",
          "uint8 immediate_or_cancel", "uint8 action",
          "uint16 cash_token_symbol", "uint16 stock_token_symbol")
          ```
      - <DATA_HASH> is the keccak256 result of the following:

        ```
        "DEx2 Order"
        <market address>(160)
        <nonce>(64)
        <expireTimeSec>(64) <amountE8>(64) <priceE8>(64) <ioc>(8) <action>(8) <pairId>(32)
        ```

- HardCancelOrder
  - mark an order as cancelled in the storage, no matter whether it is currently in the storage or not.
  - opcode: 0xDE 0x04
  - body length (in uint256): 1
  - `<nonce>(64) <traderAddr>(160) <opcode>(16)`

  - **Note**
    - No signature required. The admin keeps the right to cancel any order.

- SetFeeRates
  - set the fee rates.
  - opcode: 0xDE 0x05
  - body length (in uint256): 1
  - `<withdrawFeeRateE4>(16) <takerFeeRateE4>(16) <makerFeeRateE4>(16) <opcode>(16)`

- SetFeeRebatePercent
  - set the rebate fee percentage.
  - opcode: 0xDE 0x06
  - body length (in uint256): 1
  - `<feeRebatePercent>(8) <traderAddr>(160) <opcode>(16)`

# Free External Methods

- Free external methods can be called by either the admin or the traders, at anytime, in any order.

### Deposit
- depositEth(address traderAddr) external payable
- depositToken(address traderAddr, uint16 tokenCode, uint originalAmount) external

- **Note**
  - confirmDeposit (called by exeSequence) comes after depositEth/depositToken

### Withdraw
- withdrawEth(address traderAddr) external
- withdrawToken(address traderAddr, uint16 tokenCode) external

- **Note**
  - initiateWithdraw (called by exeSequence) comes before withdrawEth/withdrawToken
