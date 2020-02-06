A Delegate is a smart contract that sends orders based on rules. [View the code on GitHub](https://github.com/airswap/airswap-protocols/tree/master/source/delegate).

# Functions

## `constructor`

Create a new `Delegate` contract.

```java
constructor(
  ISwap delegateSwap,
  IIndexer delegateIndexer,
  address delegateContractOwner,
  address delegateTradeWallet,
  bytes2 delegateProtocol
) public
```

| Param                   | Type       | Description                                     |
| :---------------------- | :--------- | :---------------------------------------------- |
| `delegateSwap`          | `ISwap`    | Swap contract the delegate will deploy with.    |
| `delegateIndexer`       | `IIndexer` | Indexer contract the delegate will deploy with. |
| `delegateContractOwner` | `address`  | Owner of the delegate.                          |
| `delegateTradeWallet`   | `address`  | Wallet the delegate will trade from.            |
| `delegateProtocol`      | `bytes2`   | The protocol identifier for Delegate contracts. |

## `setRule`

Set a trading rule on the delegate. Delegate assumes the role of sender.
Briefly this example shows how the priceCoef and priceExp function to compute the trade quantity. This calculated price indicates the threshold price that the delegate will trade at. An order requiring the delegate to send fewer tokens than its trading rule (i.e. a better price for the delegate), will also succeed.
1 senderToken = priceCoef _ 10^(-priceExp) _ signerToken

```java
function setRule(
  address senderToken,
  address signerToken,
  uint256 maxSenderAmount,
  uint256 priceCoef,
  uint256 priceExp
) external onlyOwner
```

| Param             | Type      | Description                                                    |
| :---------------- | :-------- | :------------------------------------------------------------- |
| `senderToken`     | `address` | The token the sender would send.                               |
| `signerToken`     | `address` | The token the signer would send.                               |
| `maxSenderAmount` | `uint256` | The maximum amount of token the sender would send.             |
| `priceCoef`       | `uint256` | The coefficient of the price to indicate the whole number.     |
| `priceExp`        | `uint256` | The exponent of the price to indicate location of the decimal. |

---

A successful `setRule` will emit a `SetRule` event.

```java
event SetRule(
  address senderToken,
  address signerToken,
  uint256 maxSenderAmount,
  uint256 priceCoef,
  uint256 priceExp
);
```

| Revert Reason        | Scenario                              |
| :------------------- | :------------------------------------ |
| `INVALID_PRICE_COEF` | The priceCoef must be greater than 0. |

### Price Calculations

All amounts are in the smallest unit \(e.g. wei\), so all calculations based on price result in a whole number. For calculations that would result in a decimal, the amount is rounded in the delegate's favor. For example, a price of `5.25` and `senderAmount` of `2` results in `signerAmount` of `11` rather than `10.5`. Tokens have many decimal places so these differences are very small.

### Examples

#### Example using DAI and WETH tokens

Set a rule to send up to 100,000 DAI for WETH at 0.0032 WETH/DAI. Note that DAI has a decimal representation of 18 and WETH has a decimal representation of 18 as well. Another way to think about this is that this rule is putting 100,000 DAI up for trade in return for WETH.

Set a rule to send up to 100,000 DAI for WETH at 0.0032 WETH/DAI

```java
setRule(DAIAddress, WETHAddress, 100000000000000000000000, 32, 4)
```

Set a rule to send up to 320 WETH for DAI at 0.0032 WETH/DAI.

```java
setRule(WETHAddress, DAIAddress, 320000000000000000000, 3125, 1)
```

#### Example using AST and WETH tokens

Set a rule to send up to 5,000 AST for WETH at 0.0004 AST/WETH. Note that AST has a decimal representation of 4 and WETH has a decimal representation of 18.

```java
setRule(ASTAddress, WETHAddress, 50000000, 40000000000, 0)
```

Set a rule to send up to 2 WETH for AST at 0.0004 AST/WETH. Note that AST has a decimal representation of 4 and WETH has a decimal representation of 18.

```java
setRule(WETHAddress, ASTAddress, 2000000000000000000, 25, 12)
```

## `unsetRule`

Unset a trading rule for the delegate.

```java
function unsetRule(
  address senderToken,
  address signerToken
) external onlyOwner
```

| Param         | Type      | Description                      |
| :------------ | :-------- | :------------------------------- |
| `senderToken` | `address` | The token the sender would send. |
| `signerToken` | `address` | The token the signer would send. |

A successful `unsetRule` will emit an `UnsetRule` event.

```java
event UnsetRule(
  address senderToken,
  address signerToken
);
```

## `setRuleAndIntent`

Ssets a rule on the delegate and an intent on the indexer.

```java
function setRuleAndIntent(
  address senderToken,
  address signerToken,
  Rule calldata rule,
  uint256 amountToStake
) external onlyOwner
```

| Param           | Type      | Description                      |
| :-------------- | :-------- | :------------------------------- |
| `senderToken`   | `address` | The token the sender would send. |
| `signerToken`   | `address` | The token the signer would send. |
| `rule`          | `Rule`    | Rule to set on a delegate.       |
| `amountToStake` | `uint256` | Amount to stake for an intent.   |

A successful `setRuleAndIntent` will emit a `SetRule` event and `Stake` event. It will be an
all-or-nothing transaction.

| Revert Reason             | Scenario                                                                        |
| :------------------------ | :------------------------------------------------------------------------------ |
| `INVALID_PRICE_COEF`      | The priceCoef must be greater than 0.                                           |
| `STAKING_TRANSFER_FAILED` | The Delegate contract was not approved to transfer the staking token to itself. |
| `STAKING_RETURN_FAILED`   | The Delegate was unable to transfer remaining staked amount back.               |

```java
event SetRule(
  address indexed ruleOwner,
  address indexed senderToken,
  address indexed signerToken,
  uint256 maxSenderAmount,
  uint256 priceCoef,
  uint256 priceExp
);

event Stake(
  address indexed staker,
  address indexed signerToken,
  address indexed senderToken,
  uint256 stakeAmount
);
```

## `unsetRuleAndIntent`

Sets a rule on the delegate and an intent on the indexer.

```java
function unsetRuleAndIntent(
  address senderToken,
  address signerToken
) external onlyOwner
```

| Param         | Type      | Description                      |
| :------------ | :-------- | :------------------------------- |
| `senderToken` | `address` | The token the sender would send. |
| `signerToken` | `address` | The token the signer would send. |

A successful `unsetRuleAndIntent` will emit an `UnsetRule` event and `Unstake` event. It will be an
all-or-nothing transaction.

| Revert Reason | Scenario |
| :-------------------------- | :------------------------------------------------------------- | |
| `STAKING_RETURN_FAILED` | The Delegate was unable to transfer remaining staked amount back. |

```java
event UnsetRule(
  address indexed ruleOwner,
  address indexed senderToken,
  address indexed signerToken
);

event Unstake(
  address indexed staker,
  address indexed signerToken,
  address indexed senderToken,
  uint256 stakeAmount
);
```

## `getSignerSideQuote`

Get a quote for the signer side. Often used to get a buy price for \senderToken.

```java
function getSignerSideQuote(
  uint256 senderAmount,
  address senderToken,
  address signerToken
) external view returns (
  uint256 signerAmount
)
```

| Param          | Type      | Description                                           |
| :------------- | :-------- | :---------------------------------------------------- |
| `senderAmount` | `uint256` | The amount of ERC-20 token the sender would send.     |
| `senderToken`  | `address` | The address of an ERC-20 token the sender would send. |
| `signerToken`  | `address` | The address of an ERC-20 token the signer would send. |

## `getSenderSideQuote`

Get a quote for the sender side. Often used to get a sell price for \_signerToken.

```java
function getSenderSideQuote(
  uint256 signerAmount,
  address signerToken,
  address senderToken
) external view returns (
  uint256 senderAmount
)
```

| Param          | Type      | Description                                           |
| :------------- | :-------- | :---------------------------------------------------- |
| `signerAmount` | `uint256` | The amount of ERC-20 token the signer would send.     |
| `signerToken`  | `address` | The address of an ERC-20 token the signer would send. |
| `senderToken`  | `address` | The address of an ERC-20 token the sender would send. |

## `getMaxQuote`

Get the maximum quote from the sender.

```java
function getMaxQuote(
  address senderToken,
  address signerToken
) external view returns (
  uint256 senderAmount,
  uint256 signerAmount
)
```

| Param         | Type      | Description                                           |
| :------------ | :-------- | :---------------------------------------------------- |
| `senderToken` | `address` | The address of an ERC-20 token the sender would send. |
| `signerToken` | `address` | The address of an ERC-20 token the signer would send. |

## `provideOrder`

Provide an order to the sender for taking.

{% hint style="warning" %}
The sender specified on the order must have authorized this contract to swap on its behalf beforehand. See [Swap Contract](swap-contract.md)
{% endhint %}

```java
function provideOrder(
  Types.Order memory order
) external
```

| Param   | Type                      | Description                                                |
| :------ | :------------------------ | :--------------------------------------------------------- |
| `order` | [`Order`](types.md#order) | Order struct as specified in the `@airswap/types` package. |

---

| Revert Reason               | Scenario                                                       |
| :-------------------------- | :------------------------------------------------------------- |
| `SIGNER_MUST_BE_SENDER`     | The msg.sender is not set as the order signer.                 |
| `INVALID_SENDER_WALLET`     | The sender wallet is not set to the tradeWallet.               |
| `SIGNER_KIND_MUST_BE_ERC20` | The order.signer.kind is not ERC-20.                           |
| `SENDER_KIND_MUST_BE_ERC20` | The order.sender.kind is ERC-20.                               |
| `TOKEN_PAIR_INACTIVE`       | There is no rule set for this token pair.                      |
| `AMOUNT_EXCEEDS_MAX`        | The amount of the trade would exceed the maximum for the rule. |
| `PRICE_INCORRECT`           | The order is priced incorrectly for the rule.                  |

## `setTradeWallet`

Set a new trade wallet.

```java
function setTradeWallet(
  address newTradeWallet
) external onlyOwner
```

| Param            | Type      | Description                          |
| :--------------- | :-------- | :----------------------------------- |
| `newTradeWallet` | `address` | The address of the new trade wallet. |

| Revert Reason           | Scenario                           |
| :---------------------- | :--------------------------------- |
| `TRADE_WALLET_REQUIRED` | Trade wallet cannot be set to 0x0. |

# DelegateFactory

A DelegateFactory deploys Delegate contracts. [View the code on GitHub](https://github.com/airswap/airswap-protocols/tree/master/source/delegate-factory).

## Functions

### `constructor`

Create a new `DelegateFactory` contract.

```java
constructor(
  ISwap factorySwapContract,
  IIndexer factoryIndexerContract,
  bytes2 factoryProtocol
) public
```

| Param                    | Type     | Description                                                     |
| :----------------------- | :------- | :-------------------------------------------------------------- |
| `factorySwapContract`    | `ISwap`  | Instance of the swap contract used to settle trades.            |
| `factoryIndexerContract` | `ISwap`  | Instance of the indexer contract the delegate will deploy with. |
| `factoryProtocol`        | `bytes2` | Protocol type of the delegates the factory deploys.             |

### `createDelegate`

Create a new Delegate contract. Implements `IDelegateFactory.createDelegate`.

```java
function createDelegate(
  address delegateTradeWallet
) external returns (address delegateContractAddress)
```

| Param                 | Type      | Description                                          |
| :-------------------- | :-------- | :--------------------------------------------------- |
| `delegateTradeWallet` | `address` | Address of the wallet that holds funds to be traded. |

### `has`

Check to see whether the factory has deployed a delegate by locator. Implements `ILocatorWhitelist.has`.

```java
function has(
  bytes32 locator
) external view returns (bool)
```

| Param     | Type      | Description                                                                |
| :-------- | :-------- | :------------------------------------------------------------------------- |
| `locator` | `bytes32` | The locator in question. Expects a contract address in the first 20 bytes. |