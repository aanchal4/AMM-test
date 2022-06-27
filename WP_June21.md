<pre>    
Title:        <b>Native Automated Market Maker on XRP Ledger</b>
Revision:     <b>1</b> (2022-06-21)

<hr>Authors: <a href="mailto:amalhotra@ripple.com">Aanchal Malhotra</a>
        <a href="mailto:david@ripple.com">David J. Schwartz</a> 
Affiliation:  <a href="https://ripple.com">Ripple</a>

</pre>

# Automated Market Maker on XRP Ledger

## Abstract
The XRPL decentralized exchange (DeX) currently provides liquidity exclusively by manual market making and order books. This proposal adds non-custodial automated market maker (AMM) as a native feature to the XRPL DeX in a way that provides increased returns to those who provide liquidity for the AMM and minimizes the risk of losses due to volatility. We propose a unique mechanism where the AMM instance continuously auctions off trading advantages to arbitrageurs, charging them and adding these earnings to its capital pool. This allows liquidity providers to take a large share of the profits that would normally be taken by arbitrageurs. Additionally, the AMM instance charges a fee on the trades that changes the ratio of tokens in the instance's pools. The trading fee is added to the AMM instance's capital pool, thus adding to the liquidity providers' returns. The AMM instances also provide governance rights to the largest share holders of the AMM instance to vote on the trading fee for the instance. XRPL's AMM based DeX interacts with XRPL's limit order book based DeX, meaning that the users of AMM pools have access to all order flow and liquidity on LOB DeX, and vice versa. The transactors automatically determine whether swapping within a liquidity pool or through the order book will provide the best price for the user and execute accordingly.

## Introduction
AMMs are the agents that pool liquidity and make it available to traders according to a predetermined algorithm. This is a proposal for geometric mean market maker (GM3) as a native XRPL feature. GM3 algorithmically discovers a fair exchange value, stabilized by arbitrage, to automatically produce liquidity. On XRPL, this would provide liquidity pools between XRP and issued assets as well as between any two issued assets. Arbitrageurs play a significant role in keeping the AMM DeX in a stable state with respect to external markets. Several things that contribute to the costs that trading imposes on AMM pools are naturally much less significant on XRPL.

1. Low transaction fee: An arbitrageur only submits a transaction when the expected profit from the transaction exceeds the transaction fee. XRPL’s transaction fees are much lower than those on most DeFi chains. This benefits the AMM instance pools by narrowing the time windows in which the instance suffers decreased trading volume due to its spot-market price being off due to volatility or asymmetric trading.

2. Fast finality: Arbitrageurs take risk (for which they must be compensated at the instance pool’s expense) due to the block times. Prices may change while their transaction is in flux. XRPL has faster block times than many of the fastest competing major blockchains.

3. Canonical transaction ordering: Transactions on XRPL are canonically ordered. Other blockchains have block producers, miners, or stakers who try to extract value from arbitrage transactions by delaying, reordering, front-running, or selectively including them to extract more value from the pool and the arbitrageurs. XRPL doesn’t have this.

AMMs are more effective and lucrative when transactions execute quickly, cheaply, and fairly. This means that XRPL AMM could provide more liquidity at lower prices, yet still provide a compelling return.

On XRPL, we propose the existing **`AccountRoot`** ledger entry to represent an AMM instance with two asset pools. A new ledger entry `AMM` is also introduced to keep the additional state information of this AMM instance. New transaction type **`AMMInstanceCreate`** is used to create **`AccountRoot`** and the corresponding `AMM` ledger entries that represents a new AMM instance with two assets. This proposal allows for the creation of only one AMM instance per unique asset pair. Two new transaction types **`AMMDeposit`**, **`AMMWithdraw`**. The **`AMMDeposit`** and **`AMMWithdraw`** transactions enable adding and removing liquidity to and from the two pools of the AMM instance respectively. To enable exchanging one pool asset of the AMM instance for the other, a swap, we propose the existing `Payment` transaction. Adding liquidity to an AMM instance yields pool shares called **`LPTokens`** that are issued by the AMM instance represented as XRPL tokens. The proposal allows for both _equal-asset_ as well as _single-sided_ liquidity provision and withdrawal. The users of the instance are charged with a trading fee on the portion of the trade that changes the ratio of tokens in the AMM instance. This fee is added to the AMM instance's pools and is distributed proportionally to the `LPToken` holders upon redemption of `LPTokens`. The trading fee is a votable parameter for the `LPToken` holders. This proposal allows for the interleaved execution of the AMM trades with the existing order book based DeX on XRPL, thus ensuring aggregated liquidity and better pricing for the traders. **`LPTokens`** can also be bought, sold and can be used to make payments exactly as other issued assets/tokens on the ledger.

## Terminology

1. `Conservation Function`: We propose a weighted geometric mean market maker (GM3) for AMM price discovery:

$$C = \Gamma_{A}^{W_{A}} *  \Gamma_{B}^{W_{B}} \tag{I}$$

where,

- $\Gamma_{A}$: Current balance of token $A$ in the AMM instance pool
- $\Gamma_{B}$: Current balance of token $B$ in the AMM instance pool
- $W_{A}$: Normalized weight of token $A$ in the AMM instance pool
- $W_{B}$: Normalized weight of token $B$ in the AMM instance pool

In this proposal, the default is pools of equal value. The implicit weights are $W_{A} = W_{B} = 0.5$ [Otherwise an error]

1. `Liquidity Providers`: Liquidity providers (LPs) are the traders who add liquidity to the AMM instance's pools, thus earning shares of the pools in the form of `LPTokens`.

2. `LPTokens`: LP tokens represent the liquidity providers' shares of the AMM instance's pools. `LPTokens` are tokens on XRPL. Each `LPToken` represents a proportional share of each pool of the AMM instance. The AMM instance account _issues_ the `LPTokens` to LPs upon liquidity provision. `LPTokens` are _balanced_ in the LPs trustline upon liquidity removal.

3. `Spot-Price`: Spot-price (SP) is the ratio of the instance's pool balances. $SP_{A}^{B}$ is the spot-price of asset $A$ relative to asset $B$. $TFee$ is the trading fee paid by the trader for trades executed against the AMM instance.
   
<NOTE to self: This is the price of one unit of token A in terms of token B.>


$$SP_{A}^{B}  = \frac{\frac{\Gamma_B}{W_B}}{\frac{\Gamma_A}{W_A}} * \frac{1}{\left(1-TFee\right)} \tag{II}$$

4. `Effective-Price`: The effective price (EP) of a trade is defined as the ratio of the tokens the trader sold or swapped in (Token $B$) and the token they got in return or swapped out (Token $A$).

$$EP_{A}^{B} = \frac{\Delta_B}{\Delta_A} \tag{III}$$

<Note to self: This is the price of one unit of token A in terms of token B When we say the effwctive price of trade it always means the price of per out token in terms of in token>

5. `Slippage`: Slippage is defined as the percentage change in the effective price of a trade relative to the pre-swap spot-price.

# Creating AMM instance on XRPL

## On-Ledger Object

We propose the existing **`AccountRoot`** object together with a new ledger object **`AMM`** to represent an AMM instance. Currently, the **`AccountRoot`** object type describes a single account, its settings, and XRP balance on the XRP ledger. The **`AccountRoot`** and the **`AMM`** objects that represent an AMM instance can be created using the new **`AMMInstanceCreate`** transaction. XRP balance of the AMM instance pools is always tracked via the existing `Balance` field of the **`AccountRoot`** object. The issued asset balances of the AMM instance pools and `LPTokens` are automatically tracked via trustlines. The AMM can be traded against using the new **`AMMDeposit`** and **`AMMWithdraw`** and the existing `Payment` transactions.

Note that in the first version, we only allow for equal-weighted pools. However, the proposal is general enough to easily allow for differing weighted pools in future versions.

### AMM Ledger Entry

The unique ID of the **`AccountRoot`** object that represents an AMM instance, a.k.a **`AccountRootID`** is generated as any **`AccountRootID`** on XRPL.

The unique ID of the new **`AMM`** object , a.k.a **`AMMID`** is computed as follows:
- Calculate the `SHA512-Half` of some of the following values:
  - The `issuer` of the issued asset;
  - The `currency` of the issued asset;
  - The `issuer` of the second issued asset, if there exists one;
  - The `currency` of the second issued asset, if there exists one;
  - `XRP` if the `Balance` field exists, i.e. if one of the assets is XRP
  
The order of the fields to compute the hash is decided canonically. The **`AMMID`** associated with this **`AccountRootID`** is this hash to ensure the uniqueness of the AMM instance. The applications can look-up the **`AccountRootID`** for a specific AMM instance with this unique hash associated with the AMM account. See below for more details.

The **`AMM`** ledger entry contains the following fields for the **`AccountRoot`** object that represents the AMM instance.

1. `AccountRootID` specifies the ID of the `AccountRoot` object associated with this `AMM` ledger entry.
2. `TradingFee` specifies the fee, in basis point, to be charged to the traders for the trades executed against this AMM instance. Valid values for this field are between 0 and 70000 inclusive. A value of 1 is equivalent to 1/10 bps or 0.001%, allowing trading fee between 0% and 70%. Trading fee is a percentage of the trade size. It is charged on the asset being deposited for the `AMMDeposit` (if applicable) and `Payment` transactions and on the asset being withdrawan for the `AMMWithdraw` (if applicable) transaction. This fee is added to the AMM instance's pools and is distributed to the LPs in proportion to the `LPTokens` upon liquidity removal.

3. `AMMWeight` specifies the weight of one of the tokens of the AMM instance.
4. `VoteSlots` represents an array of `Vote` objects.
5. `AuctionSlot` represents the `Auction` object.
   
Once the **`AccountRoot`** object is created, we make sure that no further transactions can originate from this account. Conceptually, it is an account that is not owned by anyone. So every possible way of signing the transaction for this account MUST be automatically disabled.

## Transaction for creating AMM instance

We define a new transaction **`AMMInstanceCreate`** specifically for creating a new AMM instance represented by an **`AccountRoot`** object and the corresponding **`AMM`** object.

### Transaction fields for **`AMMInstanceCreate`** transaction


| Field Name        |     Required?      | JSON Type | Internal Type |
| ----------------- | :----------------: | :-------: | :-----------: |
| `TransactionType` | :heavy_check_mark: | `string`  |   `UINT16`    |


`TransactionType` specifies the new transaction type **`AMMInstanceCreate`**. The integer value is TBD.


| Field Name |     Required?      | JSON Type | Internal Type |
| ---------- | :----------------: | :-------: | :-----------: |
| `Fee`      | :heavy_check_mark: | `string`  |   `AMOUNT`    |

`Fee` specifies the integer amount of XRP, in drops, to be destroyed as a cost of creating an AMM instance. We DO NOT propose to keep a reserve for the AMM instance.

<NOTE to self: Decide on the special fee for this transaction>

| Field Name      |     Required?      |      JSON Type       | Internal Type |
| --------------- | :----------------: | :------------------: | :-----------: |
| `Asset1` | :heavy_check_mark: | `string` or `object` |   `AMOUNT`    |

`Asset1` specifies one of the pool assets (XRP or token) of the AMM instance.

| Field Name      |     Required?      | JSON Type | Internal Type |
| --------------- | :----------------: | :-------: | :-----------: |
| `Asset2` | :heavy_check_mark: | `string` or `object`  |   `AMOUNT`    |

`Asset2` specifies the other pool asset of the AMM instance.

Both `Asset1` and `Asset2` that represent XRPL tokens MUST have `value` subfields specified. The minimum amounts TBD.

| Field Name   |     Required?      | JSON Type | Internal Type |
| ------------ | :----------------: | :-------: | :-----------: |
| `TradingFee` | :heavy_check_mark: | `number` |   `UINT32`    |

`TradingFee` specifies the fee, in basis point, to be charged to the traders for the trades executed against the AMM instance. Trading fee is a percentage of the trading volume. Valid values for this field are between 0 and 70000 inclusive. A value of 1 is equivalent to 1/10 bps or 0.001%, allowing trading fee between 0% and 70%. 

The `AMMInstanceCreate` transaction MUST fail [Error code TBD] if the account issuing this transaction:
i. does not have sufficient balances, OR
ii. is NOT the `issuer` of either token AND the `RequireAuth` flag for the corresponding token is set AND the account inititaing the transaction is not authorized.

If the transaction is successful, 
i. Two new ledger entries **`AccountRoot`** and `AMM` are created.
ii. The regular key for the **`AccountRoot`** ledger entry is set to account zero, and the master key is disabled, effectively disabling all possible ways to sign transactions from this account.
iii. New trustlines are created.  
 
### Trustlines
A successful `AMMInstanceCreate` transaction will automatically create the following trustlines:

- Trustlines that track the balance(s) of the issued asset(s) of the AMM instance's pool(s) between the AMM instance account and the issuer.
- Trustlines that track the balance of `LPTokens` between the AMM instance and the account that initiated the transaction. `LPTokens` are uniquely identified by the following:
  - issuer: AMM instance **`AccountID`** and;
  - currency: the currency code for `LPTokens` is formed deterministically by concatenating the `currency` codes of the two assets of the AMM instance and `ripemd160` hashing it to 160 bits.

The `value` field is computed as follows:

$$LPTokens = \Gamma_{A}^{W_{A}} *  \Gamma_{B}^{W_{B}} $$

where,

- $\Gamma_{A}$: Balance of asset $A$
- $\Gamma_{B}$: Balance of asset $B$
- $W_{A}$ & $W_{B}$: Normalized weights of asset $A$ and asset $B$ in the pool respectively

Initially by default, $W_{A}$ = $W_{B}$ = 0.5

<NOTE to self: Address the cases when authorization behaviours change on a trustline.>

### Cost of creating AMM instance `AccountRoot` and `AMM` ledger entries

Unlike other objects in the XRPL, there is no reserve for the `AccountRoot` and `AMM` ledger entries created by an `AMMInstanceCreate` transaction. Instead there is a higher `Fee` for `AMMInstanceCreate` transaction in XRP which is burned as a special transaction cost.

### Deleting the AMM instance `AccountRoot` and `AMM` ledger entries
The `AccountRoot` and `AMM` ledger entries for unused AMM instance must be automatically deleted. An AMM instance is considered unused if no account has a trustline for the corresponding `LPTokens`. In order to avoid destroying assets of the AMM instance, the implementation of the `AMMWithdraw` transaction MUST guarantee that the AMM instance has no asset reserves if no account owns its `LPTokens`.                                                                  
## AMM trade transactions

Users can trade against specific AMM instances using the new transactions **`AMMDeposit`** and **`AMMWithdraw`**, and the existing **`Payment`** transaction.

1. `AMMDeposit`: The deposit transaction is used to add liquidity to the AMM instance pool, thus obtaining some share of the instance's pools in the form of `LPTokens`.
2. `AMMWithdraw`: The withdraw transaction is used to remove liquidity from the AMM instance pool, thus redeeming some share of the pools that one owns in the form of `LPTokens`.
3. `Payment`: The payment transaction is used to exchange one asset of the AMM instance for the other.

### AMMDeposit transaction

With `AMMDeposit` transaction, XRPL AMM allows for both:

- **all assets** liquidity provision
- **single asset** liquidity provision

#### Fields for AMMDeposit transaction

| Field Name        |     Required?      | JSON Type | Internal Type 
| ----------------- | :----------------: | :-------: | :-----------: 
| `TransactionType` | :heavy_check_mark: | `string`  |   `UINT16`    |


`TransactionType` specifies the new transaction type **`AMMDeposit`**. The integer value is TBD.


| Field Name |     Required?      | JSON Type | Internal Type |
| ---------- | :----------------: | :-------: | :-----------: |
| `AMMID` | :heavy_check_mark: | `string`  |   `UINT256`   |

`AMMID` specifies the unique **`AMMID`** of the AMM instance against which the transaction is to be executed.

| Field Name        | Required? |      JSON Type       | Internal Type |
| ----------------- | :-------: | :------------------: | :-----------: |
| `Asset1In` |           | `string` or `object` |   `AMOUNT`    |

`Asset1In` specifies one of the pools assets that the trader is willing to add. If the asset is XRP, then the `Asset1In` is a `string` specifying the number of drops. Otherwise it is an `object` with the following subfields:

| Field name |     Required?      | Description                                                                                                     |
| :--------: | :----------------: | :-------------------------------------------------------------------------------------------------------------- |
|  `issuer`  | :heavy_check_mark: | specifies the unique XRPL account address of the entity issuing the currency                                    |
| `currency` | :heavy_check_mark: | arbitrary code for currency to issue                                                                            |
|  `value`   |                    | specifies the maximum amount of this currency, in decimal representation, that the trader is willing to add |


| Field Name       | Required? | JSON Type | Internal Type |
| ---------------- | :-------: | :-------: | :-----------: |
| `Asset2In` |           | `string` or `object` |   `AMOUNT`    |

`Asset2In` specifies the details of other pool asset that the trader is willing to add.


| Field Name | Required? | JSON Type | Internal Type |
| ---------- | :-------: | :-------: | :-----------: |
| `EPrice`    |           | `string` or `object`  |   `AMOUNT`    |

`EPrice` specifies the effective-price of the token out after successful execution of the transaction. For `AMMDeposit` transaction, the token out is always `LPToken`. So:
1. If the asset being deposited is `XRP`, `EPrice` is a string
2. If the asset being deposited is other token, then `EPrice` is an `object`

Note: The relative pricing does not change in case of **all-asset** deposit transaction.
`EPrice` is an invalid field for all assets deposits. It should only be specified in case of single sided deposits.


| Field Name | Required? | JSON Type | Internal Type |
| ---------- | :-------: | :-------: | :-----------: |
| `LPTokens` |           | `object`  |   `AMOUNT`    |

`LPTokens` specifies the amount of shares of the AMM instance pools that the trader wants to buy.

Let the following represent the pool composition of AMM instance before trade:

- $\Gamma_{A}$: Current balance of asset $A$
- $\Gamma_{B}$: Current balance of asset $B$
- $\Gamma_{LPTokens}$: Current balance of outstanding `LPTokens` issued by the AMM instance

Let the following represent the assets being deposited as a part of `AMMDeposit` transaction and the corresponding `LPTokens` being issued by the AMM instance.

- $\Delta_{A}$: Balance of asset $A$ being added
- $\Delta_{B}$: Balance of asset $B$ being added
- $\Delta_{LPTokens}$: Balance of `LPTokens` issued to the LP after a successful `AMMDeposit` transaction

And let $TFee$ represent the trading fee paid by the account that initiated the transaction to the AMM instance's account. This fee is added to the corresponding AMM instance pool and is proportionally divided among the `LPToken` holders.

### All assets deposit (pure liquidity provision)

If the trader deposits proportional values of both assets without changing their relative pricing, no trading fee is charged on the transaction. If $A$ and $B$ are the assets being deposited in return for $\Delta_{LPTokens}$, then

$$\Delta_{A} = \left(\frac{\Delta_{LPTokens}}{\Gamma_{LPTokens}}\right) * \Gamma_{A} \tag{1}$$

AND

$$\Delta_{B} = \left(\frac{\Delta_{LPTokens}}{\Gamma_{LPTokens}}\right) * \Gamma_{B} \tag{2}$$

Following is the updated pool composition of the AMM instance after successful transaction:

- $\Gamma_{A} = \Gamma_{A} + \Delta_{A}$: Current new balance of asset $A$
- $\Gamma_{B} = \Gamma_{B} + \Delta_{B}$: Current new balance of asset $B$
- $\Gamma_{LPTokens} = \Gamma_{LPTokens} + \Delta_{LPTokens}$: Current new balance of outstanding `LPTokens`

### Single asset deposit

Let $B$ be the only asset being deposited in return for $\Delta_{LPTokens}$, then this is executed as a combination of _equal asset_ deposit and a _swap_ transaction. The trading fee is only charged on the _amount_ of asset being swapped.

$$\Delta_{LPTokens} = \Gamma_{LPTokens} * \left[ \left(1+\frac{\Delta_{B} - TFee * \left(1 - W_{B}\right) * \Delta_{B}}{\Gamma_{B}}\right)^{W_{B}} - 1 \right] \tag{3}$$

Similarly,
$$\Delta_B = \left[\frac{\left(\left(\frac{\Delta_{LPTokens}}{\Gamma_{LPTokens}} + 1\right)^{\frac{1}{W_{B}}} - 1\right)}{\left(1 - TFee * (1 - W_B)\right)}\right] * \Gamma_B \tag{4}$$

Following is the updated pool composition of the AMM instance after successful trade:

- $\Gamma_{B} = \Gamma_{B} + \Delta_{B}$: Current new balance of asset $B$
- $\Gamma_{LPTokens} = \Gamma_{LPTokens} + \Delta_{LPTokens}$: Current new balance of outstanding `LPTokens`


### Specifying different parameters

The proposal allows for traders to specify different combinations of the abovementioned fields for `AMMDeposit` transaction. The implementation will determine the best possible suboperations based on trader's specifications. Following are the recommended valid combinations. Other invalid combinations may result in the failure of transaction.

- `LPTokens`
- `Asset1In`
- `Asset1In` and `Asset2In`
- `Asset1In` and `LPTokens`
- `Asset1In` and `EPrice`

Details for above combinations:

1. Fields specified: `LPTokens`

Such a transaction assumes proportional deposit of pools assets in exchange for the specified amount of `LPTokens` of the AMM instance.


2. Fields specified: `Asset1In`

Such a transaction assumes single asset deposit of the amount of asset specified by `Asset1In`. This is essentially an _equal_ asset deposit and a _swap_.

Note: If the asset to be deposited is a token, specifying the `value` field is required, else the transaction will fail.

3. Fields specified: `Asset1In` and `Asset2In`

Such a transaction assumes proportional deposit of pool assets with the constraints on the maximum amount of each asset that the trader is willing to deposit.


4. Fields specified: `Asset1In` and `LPTokens`

Such a transaction assumes that a single asset `asset1` is deposited to obtain some share of the AMM instance's pools represented by amount of `LPTokens`. Since adding liquidity to the pool with one asset changes the ratio of the assets in the two pools, thus changing the relative pricing, trading fee is charged on the amount of the deposited asset that causes this change.

5. Fields specified: `Asset1In` and `EPrice`

Such a transaction assumes single asset deposit with the following two constraints:

a. amount of asset1 if specified in `Asset1In` specifies the maximum amount of asset1 that the trader is willing to deposit

b. The effective-price of the `LPTokens` traded out does not exceed the specified `EPrice`


Following updates after a successful `AMMDeposit` transaction:

- The deposited asset, if XRP, is transferred from the account that inititaed the transaction to the AMM instance account, thus changing the `Balance` field of each account
- The deposited asset, if tokens, are balanced between the AMM account and the issuer account trustline.
- The `LPTokens` ~ $\Delta_{LPTokens}$ are issued by the AMM instance account to the account that initiated the transaction and a new trustline is created, if there does not exist one.
- The pool composition is updated. Note that the conservation function is not preserved in case of liquidity provision.

### **`AMMWithdraw`** transaction

With `AMMWithdraw` transaction, XRPL AMM allows for both the following:

- **all assets** liquidity withdrawal
- **single asset** liquidity withdrawal
  
#### Fields
| Field Name        |     Required?      | JSON Type | Internal Type 
| ----------------- | :----------------: | :-------: | :-----------: 
| `TransactionType` | :heavy_check_mark: | `string`  |   `UINT16`    |


`TransactionType` specifies the new transaction type **`AMMWithdraw`**. The integer value is TBD.


| Field Name |     Required?      | JSON Type | Internal Type |
| ---------- | :----------------: | :-------: | :-----------: |
| `AMMID` | :heavy_check_mark: | `string`  |   `UINT256`   |

`AMMID` specifies the unique **`AMMID`** of the AMM instance against which the transaction is to be executed.

| Field Name         |     Required?      |      JSON Type       | Internal Type |
| ------------------ | :----------------: | :------------------: | :-----------: |
| `Asset1Out` |  | `object` or `string` |   `AMOUNT`    |

`Asset1Out` specifies one of the pools assets that the trader wants to remove. If the asset is XRP, then the `Asset1Out` is a `string` specifying the number of drops. Otherwise it is an `object` with the following subfields:

| Field name |     Required?      | Description                                                                       |
| :--------: | :----------------: | :-------------------------------------------------------------------------------- |
|  `issuer`  | :heavy_check_mark: | specifies the XRPL address of the issuer of the currency                          |
| `currency` | :heavy_check_mark: | specifies the currency code of the issued currency                                |
|  `value`   |                    | specifies the minimum amount of this asset that the trader is willing to withdraw. |

| Field Name        | Required? | JSON Type | Internal Type |
| ----------------- | :-------: | :-------: | :-----------: |
| `Asset2Out` |           | `string` or `object` |   `AMOUNT`    |

`Asset2Out` specifies the other asset that the trader wants to remove.

| Field Name | Required? | JSON Type | Internal Type |
| ---------- | :-------: | :-------: | :-----------: |
| `EPrice`    |           | `object` |   `AMOUNT`    |

`EPrice` specifies the effective-price of the token out after successful execution of the transaction. For `AMMWithdraw` transaction, the out is either XRP or issued token. The asset in is always `LPToken`. So `EPrice` is always an `object`.

Note: The relative pricing does not change in case of **all-asset** withdrawal. `EPrice` is an invalid field for all assets deposits. It should only be specified in case of single sided deposits.

| Field Name | Required? | JSON Type | Internal Type |
| ---------- | :-------: | :-------: | :-----------: |
| `LPTokens` |           | `object`  |   `AMOUNT`    |

`LPTokens` specifies the amount of shares of the AMM instance pools that the trader wants to redeem or trade in.

Let following represent the pool composition of the AMM instance before withdrawal:

- $\Gamma_{A}$: Current balance of asset $A$ 
- $\Gamma_{B}$: Current balance of asset $B$
- $\Gamma_{LPTokens}$: Current balance of outstanding `LPTokens` issued by the AMM instance


Let following represent the assets being withdrawn as a part of `AMMWithdraw` transaction and the corresponding `LPTokens` being redeemed.

- $\Delta_A$: Balance of asset $A$ being withdrawn
- $\Delta_B$: Balance of asset $B$ being withdrawn
- $\Delta_{LPTokens}$: Balance of `LPTokens` being redeemed

Let $TFee$ represent the trading fee paid by the trader to the AMM instance, if applicable. The fee is added to the appropriate pool and is distributed proportionally among the `LPToken` holders upon liquidity withdrawal.

### All assets withdrawal (Pure liquidity withdrawal)

If the trader withdraws proportional values of both assets without changing their relative pricing, no trading fee is charged on the transaction. If $A$ and $B$ are the assets being withdrawn by redeeming $\Delta_{LPTokens}$, then

$$\Delta_A = \left(\frac{\Delta_{LPTokens}}{\Gamma_{LPTokens}}\right) * \Gamma_A \tag{5}$$

AND

$$\Delta_B = \left(\frac{\Delta_{LPTokens}}{\Gamma_{LPTokens}}\right) * \Gamma_B \tag{6}$$

Following is the updated pool composition of the AMM instance after successful trade:

- $\Gamma_{A} = \Gamma_{A} - \Delta_{A}$: Current new balance of asset $A$
- $\Gamma_{B} = \Gamma_{B} - \Delta_{B}$: Current new balance of asset $B$
- $\Gamma_{LPTokens} = \Gamma_{LPTokens} - \Delta_{LPTokens}$: Current new balance of outstanding `LPTokens`

### Single asset withdrawal

Single asset withdrawal can be conceptualized as two subtrades of _equal_ asset withdrawal and a _swap_. Let asset $B$ be the only asset being withdrawn to redeem $\Delta_{LPTokens}$, then


$$\Delta_{LPTokens} = \Gamma_{LPTokens}* \left[1- \left(1- \frac{\Delta_B}{\Gamma_B*(1-(1- W_B )*TFee)}\right)^{W_{B}}\right] \tag{7}$$

Similarly,
$$\Delta_B = \Gamma_B *  \left[ 1-  \left(1- \frac{\Delta_{LPTokens}}{\Gamma_{LPTokens}} \right)^ \frac{1}{W_{B}}   \right] *  \left[1-\left(1-W_B \right)*TFee \right]  \tag{8}$$

Following is the updated pool composition of the AMM instance after successful trade:

- $\Gamma_B = \Gamma_B - \Delta_{B}$: Current new balance of asset $B$
- $\Gamma_{LPTokens} = \Gamma_{LPTokens} - \Delta_{LPTokens}$: Current new balance of outstanding `LPTokens`

### Specifying different parameters

The proposal allows for traders to specify different combinations of the abovementioned fields for `AMMWithdraw` transaction. The implementation will figure out the best possible operations based on trader's specifications. Here are the recommended possible combinations. Other invalid combinations may result in the failure of transaction.

- `LPTokens`
- `Asset1Out`
- `Asset1Out` and `Asset2Out`
- `Asset1Out` and `LPTokens`
- `Asset1Out` and `EPrice`

Implementation details for the above combinations:

1. Fields specified: `LPTokens`

Such a transaction assumes proportional withdrawal of pool assets for the amount of `LPTokens`. Since withdrawing assets proportionally from the AMM instance pools does not change the ratio of the two assets in the pool and thus does not affect the relative pricing, trading fee is not charged on such a transaction.

2. Fields specified: `Asset1Out`

Such a transaction assumes withdrawal of single asset equivalent to the amount specified in `Asset1Out`

3. Fields specified: `Asset1Out` and `Asset2Out`

Such a transaction assumes all assets withdrawal with the constraints on the maximum amount of each asset that the trader is willing to withdraw.

4. Fields specified: `Asset1Out` and `LPTokens`

Such a transaction assumes withdrawal of single asset specified in `Asset1Out` proportional to the share represented by the amount of `LPTokens`. Since a single sided withdrawal changes the ratio of the two assets in the AMM instance pools, thus changing their relative pricing, trading fee is charged on the amount of asset1 that causes that change.


5. Fields specified: `Asset1Out` and `EPrice`

<NOTE to self: This is like saying I dont want to pay more than EPrice for AssetOut>

Such a transaction assumes withdrawal of single asset with the following constraints:

a. amount of asset1 if specified in `Asset1Out` specifies the minimum amount of asset1 that the trader is willing to withdraw

b. The effective price of asset traded out doesnot exceed the amount specified in `EPrice`

Following updates after a successful transaction:

- The withdrawn asset, if XRP, is transferred from AMM instance account to the account that initiated the transaction, thus changing the `Balance` field of each account
- The withdrawn asset, if token, is balanced between the AMM instance account and the issuer account.
- The `LPTokens` ~ $\Delta_{LPTokens}$ are balanced between the AMM instance account and the account that initiated the transaction.
- The pool composition is updated. Conservation function is not preserved in case of liquidity withdrawal (and is not expected to.)

## **`AMM Swap`**

In order to exchange one asset of the AMM instance's pools for another, we propose to use the existing `Payment` transaction.

Let the following represent the pool composition of the AMM instance before a swap.

- $\Gamma_{A}$: Current balance of asset $A$ in the pool
- $\Gamma_{B}$: Current balance of asset $B$ in the pool

Let the following represent the balances of assets $B$ and $A$ being swapped in and out of the AMM instance pools respectively with `Payment` transaction.

- $\Delta_{A}$: Balance of asset $A$ being swapped out of the AMM instance's pool
- $\Delta_{B}$: Balance of asset $B$ being swapped into the AMM instance's pool


We can compute $\Delta_{A}$, given $\Delta_{B}$ and $TFee$ as follows:

$$\Delta_{A} = \Gamma_{A}  *    \left[1 – \left(\frac{\Gamma_{B}}{\Gamma_{B}+ \Delta_{B} * \left(1-TFee\right)}\right)^\frac{W_{B}}{W_{A}}\right] \tag{9}$$

Similarly, we can compute $\Delta_{B}$, given $\Delta_{A}$ and $TFee$ as follows:

$$\Delta_{B}= \Gamma_{B} *\left[  \left( \frac{\Gamma_{A}}{\Gamma_{A}- \Delta_{A}} \right) ^  \frac{W_A}{W_B}-1     \right]*  \frac{1}{1-TFee} \tag{10}$$

To change the spot-price of token $A$ traded out relative to token $B$ traded into the pool from $SP_{A}^{B}$ to $SP_{A}^{'B}$, required $\Delta_{B}$ can be computed as follows:

$$\Delta_B = \Gamma_{B} *  \left[ \left(\frac{SP_{A}^{'B}}{SP_{A}^{B}}\right)^{\left(\frac{W_A}{W_A+ W_B}\right) } - 1\right] \tag{11}$$

We can compute the average slippage $S(\Delta_B)$ for the token traded in as follows:

$$S(\Delta_B) = SS_B * \Delta_B \tag{12}$$

where $SS_B$, the slippage slope is the derivative of the slippage when the traded amount tends to zero.

$$SS_B  = \frac{\left(1-TFee\right) * \left(W_B+W_A\right)}{2*\Gamma_B*W_A } \tag{13}$$

We can compute the average slippage $S(\Delta_A)$ for the token traded out as follows:
  $$S(\Delta_A) = SS_{A} * \Delta_{A} \tag{14}$$

where $SS_A$, the slippage slope is the derivative of the slippage when the traded amount tends to zero.
$$ SS_{A} = \frac{W_{A} + W_{B}}{2 * \Gamma_{B} * W_{A}} \tag{15}$$

The following is the updated pool composition after a successful transaction:

- $\Gamma_{A}$ = $\Gamma_{A}$ - $\Delta{A}$: Current new balance of asset $A$
- $\Gamma_{B}$ = $\Gamma_{B}$ + $\Delta_{B}$: Current new balance of asset $B$

Note that the conservation function is preserved, however the ratio of the two assets and hence their relative pricing changes after a successful swap.

Following updates after a successful transaction:

- The swapped asset, if XRP, is transferred from AMM instance account to the account that initiated the transaction or vice-versa, thus changing the `Balance` field of each account
- The swapped asset, if token, is balanced between the AMM instance account and the issuer account.
- The pool composition is updated.

## **`Payment`** transaction for AMM swap
An XRPL `Payment` transaction represents a transfer of value from one account to another. This transaction type can be used for several types of payments. One can accomplish an equivalent of a swap with the `Payment` transaction. Following are the relevant fields of a `Payment` transaction.

| Field Name | Required? | JSON Type | Internal Type |
| ---------- | :-------: | :-------: | :-----------: |
| `Amount`    |           | `Currency Amount` |   `AMOUNT`    |

`Amount` field is used to specify the amount of currency that the trader wants to deliver to a destination account. For an AMM swap, this is the amount of currency that the trader wants to swap out of the pool.

| Field Name | Required? | JSON Type | Internal Type |
| ---------- | :-------: | :-------: | :-----------: |
| `Destination`    |           | `String` |   `AccountID`    |

`Destination` field is used to specify the account that the trader wants to deliver the currency to. This should be either the same account as the transaction or any other account that the trader wishes to send this currency amount to.

| Field Name | Required? | JSON Type | Internal Type |
| ---------- | :-------: | :-------: | :-----------: |
| `SendMax`    |           | `Currency Amount` |   `AMOUNT`    |

`SendMax` field is used to specify the maximum amount of currency that the trader is willing to send from the account issuing the transaction. For an AMM swap, this is the amount of currency that the trader is willing to swap into the pool.

### Flags 
The `tfLimitQuality` flag is used to specify the **quality** of trade. This is defined as the ratio of the amount in to amount out. In other words, it is the ratio of amounts in `Amount` and `SendMax` fields of a `Payment` transaction.

For an AMM swap, if a trader wants to buy (swap out) one unit of currency specified in `Amount` field for not more than `X` units per currency specified in `SendMax`, the trader would use the following equation:
$$ SendMax = X * Amount$$

where `X` is equal to the `LimitSpotPrice` for the trade, i.e. the threshold on the spot-price of asset out after trade.

## Governance: Trading Fee Voting Mechanism
This proposal allows for the `TradingFee` of the AMM instance be a votable parameter. Any account that holds the corresponding `LPTokens` can cast a vote using the new `AMMVote` transaction. 

We introduce a new field `VoteSlots` associated with each AMM instance in the **`AMM`** ledger object. The `VoteSlots` field keeps a track of upto eight active votes for the instance.

| Field Name | Required? | JSON Type | Internal Type |
| ---------- | :-------: | :-------: | :-----------: |
| `VoteSlots`    |           | `array` |   `ARRAY`    |

`VoteSlots` is an array of `VoteEntry` objects representing the LPs and their vote on the `TradingFee` for this AMM instance.

### Vote Entry object
Each member of the `VoteSlots` field is an object that describes the vote for the trading fee by the LP of that instance. A `VoteEntry` has the following fields:

| Field Name | Required? | JSON Type | Internal Type |
| ---------- | :-------: | :-------: | :-----------: |
| `Account`    |           | `string` |   `AccountID`    |

`Account` specifies the XRPL address of the LP.

| Field Name | Required? | JSON Type | Internal Type |
| ---------- | :-------: | :-------: | :-----------: |
| `FeeVal`    |           | `number` |   `UINT32`    |

`FeeVal` specifies the fee, in basis point. Valid values for this field are between 0 and 70000 inclusive. A value of 1 is equivalent to 1/10 bps or 0.001%, allowing trading fee between 0% and 70%.

| Field Name | Required? | JSON Type | Internal Type |
| ---------- | :-------: | :-------: | :-----------: |
| `VoteWeight`    |           | `number` |   `UINT32`    |

`VoteWeight` specifies the `LPTokens` owned by the account that issued the transaction. It is specified in basis points. Valid values for this field are between 0 and 100000. A value of 1 is equivalent to 1/10 bps or 0.001%, allowing the percentage ownership in the instance between 0% and 100%.


`TradingFee` for the AMM instance is computed as the weighted mean of all the current votes in the `VoteSlots` field.

$$TradingFee = \frac{\sum_{i=1}^{8}(w_i * fee_i)}{\sum_{i=1}^{8}(w_i)}$$

where $w_i$ and $fee_i$ represents the `VoteWeight` and the `FeeVal` of the corresponding `VoteEntry`.

## `AMMVote` transaction

We introduce the new `AMMVote` transaction. Any XRPL account that holds `LPTokens` for an AMM instance may submit this transaction to vote for the trading fee for that instance. 

### Fields
| Field Name | Required? | JSON Type | Internal Type |
| ---------- | :-------: | :-------: | :-----------: |
| `Account`    |           | `string` |   `Account ID`    |

`Account` specifies the XRPL account that submits the transaction.

| Field Name        |     Required?      | JSON Type | Internal Type |
| ----------------- | :----------------: | :-------: | :-----------: |
| `TransactionType` | :heavy_check_mark: | `string`  |   `UINT16`    |


`TransactionType` specifies the new transaction type **`AMMVote`**. The integer value is TBD.

| Field Name | Required? | JSON Type | Internal Type |
| ---------- | :-------: | :-------: | :-----------: |
| `FeeVal`    |           | `number` |   `UINT32`    |

`FeeVal` specifies the fee, in basis point. Valid values for this field are between 0 and 70000 inclusive. A value of 1 is equivalent to 1/10 bps or 0.001%, allowing trading fee between 0% and 70%.


## How does `AMMVote` transaction work?
This `AMMVote` transaction will _always_ check if all the `VoteEntry` objects in the `VoteSlots` array are up-to-date, i.e. check if there is a change in the number of `LPTokens` owned by an account in the `VoteEntry` and do the following:
1. If one or more accounts in the `VoteEntry` does not hold the `LPTokens` for this AMM instance any more, then remove that `VoteEntry` object from the `VoteSlots` array or If the number of `LPTokens` held by one or more account in the `VoteEntry` has changed, 
   - recompute the weights of all the votes and readjust them. Also recompute the `TradingFee` for the AMM instance and update it in the `AMM` object of that instance.
 
2. Check the `LPTokens` held by the account submitting the `AMMVote` transaction. If the account does not hold any `LPTokens`, then the transaction fails with an error code (TBD). Otherwise, there are following cases: 
    - If there are fewer than eight `VoteEntry` objects in the updated `VoteSlots` array, then:
        a. Compute the weight of the current vote, add the computed weight of that vote as `VoteWeight` field to the object and readjust in the object.
        b. Add that `FeeVal` of the current vote to the `VoteEntry` object.
        c. Compute the `TradingFee` for the AMM instance as described above and update the `TradingFee` field value in the `AMM` object of the AMM instance.
        
    - If all eight `VoteEntry` slots are full, but this account holds more `LPTokens`, i.e. higher `VoteWeight` than the `VoteEntry` object with the lowest `VoteWeight`, this vote will replace the `VoteEntry` with the lowest `VoteWeight`. (Followed by same steps as above.)
  
    - If the account that submitted the transaction already holds a vote, then update that `VoteEntry` and `TradingFee` based on the transaction fields.

## Continuous Auction Mechanism
We introduce a novel mechanism for an AMM instance to auction-off the trading advantages to users (arbitrageurs) at a **discounted** `TradingFee` for a 24 hour slot. Any account that owns corresponding `LPTokens` can bid for the auction slot of that AMM instance. This is a continuous auction, any account can bid for the slot any time. Part of the proceeds from the auction, i.e. `LPTokens` are refunded to the current slot-holder computed on a pro rata basis. Remaining part of the proceeds - in the units of `LPTokens`- is burnt, thus effectively increasing the LPs expected returns. 

## Mechanics
The bid price to purchase the slot must adjust dynamically to function as an auction. If the slot is full, i.e. if an account owns the auction slot, the price should drop as the slot gets older. 

To buy a slot when the slot is full, one must pay more (per unit time) than the value at which the slot was bought. The previous slot holder loses their slot, but they receive a pro-rata refund (for their remaining time) and the rest goes to the pool (LPs).

The complex mechanism is how the price to buy the slot should drop as the occupied slot gets older. We introduce the slot price-schedule algorithm to answer the following questions:
1. How is the additional price one must pay per unit time to replace the holder of an existing slot determined?
2. How is the additional amount shared between the pool and the account losing its slot?
3. What is the amount per unit time for a slot when the slot is empty?

We introduce a new object field `AuctionSlot` in the `AMM` object associated with each AMM instance. The `AuctionSlot` field has the following subfields: 

| Field Name | Required? | JSON Type | Internal Type |
| ---------- | :-------: | :-------: | :-----------: |
| `Account`    |           | `string` |   `AccountID`    |

`Account` represents the XRPL account that currently owns the auction slot.

| Field Name | Required? | JSON Type | Internal Type |
| ---------- | :-------: | :-------: | :-----------: |
| `TimeStamp`    |           | `string` |   `UINT32`    |

`TimeStamp` represents the number of seconds since the Ripple epoch. This marks the beginning of the time when slot was bought.

| Field Name | Required? | JSON Type | Internal Type |
| ---------- | :-------: | :-------: | :-----------: |
| `DiscountedFee`    |           | `string` |   `UINT32`    |

`DiscountedFee` represents the `TradingFee` to be charged to this account for trading against the AMM instance. By default it is 0.

| Field Name | Required? | JSON Type | Internal Type |
| ---------- | :-------: | :-------: | :-----------: |
| `Price`    |           | `string` |   `AMOUNT`    |

`Price` represents the price paid for the slot specified in units of `LPTokens` of the AMM instance.

### `AMMBid` transaction
We introduce a new transaction `AMMBid` to place a bid for the auction slot. The transaction may have the following optional and required fields:

| Field Name | Required? | JSON Type | Internal Type |
| ---------- | :-------: | :-------: | :-----------: |
| `Account`    |  :heavy_check_mark: | `string` |   `AccountID`    |

`Account` represents the XRPL account that submits the transaction.

| Field Name        |     Required?      | JSON Type | Internal Type |
| ----------------- | :----------------: | :-------: | :-----------: |
| `TransactionType` | :heavy_check_mark: | `string`  |   `UINT16`    |

`TransactionType` specifies the new transaction type **`AMMBid`**. The integer value is TBD.

| Field Name | Required? | JSON Type | Internal Type |
| ---------- | :-------: | :-------: | :-----------: |
| `MinSlotPrice`    |           | `string` |   `Amount`    |

`MinSlotPrice` represents the minimum price that the auctioneer wants to pay for the slot. It is specified in units of `LPTokens`

| Field Name | Required? | JSON Type | Internal Type |
| ---------- | :-------: | :-------: | :-----------: |
| `AuthAccounts`    |           | `array` |   `Array`    |

`AuthAccounts` represents an array of XRPL account IDs that are authorized to trade at the discounted fee against the AMM instance. The proposal allows for upto a maximum of four accounts.
 
### Slot price scheduling algorithm
The minimum price to buy a slot is called **MinSlotPrice**. 
Total slot time of 24 hours is divided into 20 equal intervals. An auction slot can be in one of the following states at any given time:

State 1. Empty - no account currently holds the slot. 

State 2. Occupied - an account owns the slot with at least 5% of the remaining slot time (i.e. in one of the 1-19 intervals). 

State 3. Dead - an account owns the slot with less than 5% of the remaining time (i.e, in the last interval)  

The slot-holder owns the slot privileges when in State 2 or State 3.

#### Slot Pricing
I. Slot state - 1 or 3 

Slot Price = **MinSlotPrice** 

II. Slot state - 2

Let the price at which the slot is bought be `X` - specified in `Amount` of `LPTokens`. And let `x` represent the fraction of remaining slot time for the current slot-holder. The algorithm to compute the minimum bid price of the slot at any given time enforces the following rules:
   
1. The minimum bid price of the slot in the first interval is:
  $$X + 5\% * X$$
2.  The slot price decays exponentially over time. However, we also want the price to decay very very slowly for most of the time intervals (~95%) and instantly drop to the **MinSlotPrice** as the slot gets closer to the expiration time (~5%). We choose a heuristic function that produces this behavior. The following equation determines the minimum bid price of the slot at any time:
$$ f(x) = X * 1.05*(1-x^{60}) + min\_slot\_price$$ 
Notice that the slot price approaches the **MinSlotPrice** as the slot approaches expiration (i.e., as `x` approaches 0).

3. The revenue from a successful `AMMBid` transaction is split between the current slot-holder and the pool. We propose to _always_ refund the current slot-holder of the remaining value of the slot computed from the price at which they bought the slot. 
$$ f(x) = x*X$$ 

The remaining `LPTokens` are burnt/deleted, effectively increasing the LPs share in the pool. 

### How does AMMBid transaction work?
Upon receiving an **`AMMBid`** transaction, the slot price is computed as follows:

I. Check the state of auction slot. If the slot is in state 1, then:
    
        a. Compute the slot price as stated in I in the above section. Let it be `Y`.
        b. Check for `MinSlotPrice` field in the transaction.
       
          1. If it doesnot exist or if `MinSlotPrice` <= `Y`, then the slot price is `Y`.
          2. Otherwise: the slot price is `MinSlotPrice` 

II. Otherwise if the slot is in state 2 or 3, then:


## Error Codes
TBD

## Security Concerns
None.

## Implementation
In progress.