
<pre>
Title:       <b>Permissioned DEXes</b>
Revision:    <b>1</b> (2024-07-01)

Author:      <a href="mailto:mvadari@ripple.com">Mayukha Vadari</a>

Affiliation: <a href="https://ripple.com">Ripple</a>
</pre>

# Permissioned DEXes

## Abstract

(copied + rewritten from PRD)

Decentralized Exchanges (DEXes) are revolutionizing finance due to their ability to offer significant benefits such as intermediary elimination, lower transaction costs, enhanced security, and user asset custody. These advantages align perfectly with the growing demand for efficient financial systems. However, a major hurdle hinders wider adoption by traditional institutions: the anonymity of a DEX makes it difficult to comply with Anti-Money Laundering (AML) and Know Your Customer (KYC) regulations.

This challenge highlights a critical need for the development of a permissioned system within DEXes. Such a system would allow institutions to adhere to regulations while still benefiting from the core advantages of blockchain technology.

This proposal introduces a permissioned DEX system for the XRPL. By integrating permissioning features directly within the DEX protocol, regulated financial institutions gain the ability to participate in the XRPL's DEX while still adhering to their compliance requirements. This approach avoids the drawbacks of isolated, permissioned tokens or private blockchains, ensuring a vibrant and liquid marketplace that facilitates seamless arbitrage opportunities. Ultimately, this permissioned DEX system paves the way for wider institutional adoption of XRPL, fostering a more inclusive and efficient financial landscape.

## 1. Overview

This proposal builds on top of [XLS-70d](https://github.com/XRPLF/XRPL-Standards/discussions/202), as credentials are needed for permissioning.

We propose:
* Creating a `PermissionedDomain` ledger object.
* Creating a `PermissionedDomainSet` transaction.
* Creating a `PermissionedDomainDelete` transaction.
* Modifying the `Offer` ledger object.
* Modifying the `OfferCreate` transaction.
* Modifying the `Payment` transaction.

This feature will require an amendment, tentatively titled `featurePermissionedDEX`.

### 1.1. Background: The Current State of Permissioning and Compliance on the XRPL

* Goal: Compliance by Design
* Deposit Auth
* Authorized Trustlines
* Clawback
* Freezing
* Multisign
* Payment paths
* Why all this isn't enough - DEX activity

### 1.2. Terminology

* **Offer Crossing**: Two offers **cross** if one is selling a token at a price that's actually lower than the price the other is offering to buy it at.
* **Offer Filling**: Two offers **fill** each other if the trade executes and the sale goes through. Offers can be partially filled, based on the flags (settings) of the offers.
* **Domain**: A collection of rules indicating what accounts and trades may be a part of it. This spec includes credential-gating and token-gating, but more options could be added in the future.
* **Domain Rules**: The set of rules that govern a domain, i.e. the credentials and tokens it accepts.
* **Domain Owner**: The account that created a domain, and is the only one that can modify its rules or delete it.
* **Domain Member**: An account that satisfies the rules of the domain (i.e. has one of the credentials that are accepted by the domain). There is no explicit joining step; as long as the account has a valid credential, it is a member.
* **Permissioned DEX**: The subset of the DEX that operates within the rules of a specific domain.
* **Open DEX**: The unpermissioned DEX that has no restrictions.
* **Permissioned Offer/Payment**: An offer/cross-currency payment that can only be filled by offers that are a part of a specific domain.
* **Open Offer/Payment** or **Unpermissioned Offer/Payment**: An offer/cross-currency payment that is able to trade on the open DEX or potentially in permissioned DEXes, but doesn't have any restrictions itself.
* **Valid Domain Offer/Payment**: An offer/cross-currency payment that satisfies the rules of a domain (i.e. the account is a domain member, and the tokens in the offer are accepted). This can be a permissioned _or_ open offer.

### 1.3. Basic Flow

#### 1.3.1. Initial setup
* Owen, a domain owner, creates his domain with a set of KYC credentials and allowed tokens (including a certain USD and EUR).
* Tracy, a trader, is operating in Owen's regulatory environment and has strict regulatory requirements - she cannot receive liquidity from non-KYCed accounts. She has one of Owen's accepted KYC credentials, and will only be placing permissioned offers.
* Marko, a market maker, wants to arbitrage offers in Owen's domain, as there is often a significant price difference inside and outside. He obtains one of the KYC credentials that Owen's domain accepts. However, he will not be placing permissioned offers, only open offers, to access as much liquidity as possible.

#### 1.3.2. Trading Scenario 1

* Tracy places a permissioned offer on the XRP-USD orderbook. There are no other offers on that orderbook that are a part of the domain.
* Marko notices that there is a significant price difference between Tracy's offer and the rest of the orderbook. He now submits an open offer to cross and fill Tracy's offer.

#### 1.3.3. Trading Scenario 2
* Marko has placed open offers on the XRP-EUR orderbook.
* Tracy places an offer on the XRP-EUR orderbook that crosses one of Marko's offers. Her offer is filled by his.

### 1.4. How Domains Work

A permissioned offer will only be filled by valid domain offers.

An open offer can be filled by any offer on the open DEX, but can _also_ be filled by any permissioned offer, if the open offer is a valid domain offer.

The fact that traders don't need to place a permissioned offer in order to fill offers in that domain enables arbitrage and mixing of liquidity much more easily, as traders do not have to place multiple offers in multiple domains. It also allows traders to avoid walled gardens of liquidity.

## 2. On-Ledger Object: `PermissionedDomain`

This object represents a DEX domain.

### 2.1. Fields

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`LedgerIndex`| ✔️|`string`|`Hash256`|The unique ID of the ledger object.|
|`LedgerEntryType`| ✔️|`string`|`UInt16`|The ledger object's type (`PermissionedDomain`).|
|`Owner`| ✔️|`string`|`AccountID`|The account that controls the settings of the domain.|
|`Sequence`|✔️|`number`|`UInt32`|The `Sequence` value of the `PermissionedDomainSet` transaction that created this offer. Used in combination with the `Account` to identify this offer.|
|`AcceptedCredentials`| |`array`|`STArray`|The credentials that are accepted by the domain. Ownership of one of these credentials automatically makes you a member of the domain.|
|`AcceptedTokens`| |`array`|`STArray`|The tokens that are allowed by the domain.|

#### 2.1.1. `LedgerIndex`

The ID of this object will be a hash that incorporates the `Owner` and `Sequence` fields, combined with a unique space key for `PermissionedDomain` objects, which will be defined during implementation.

This value will be used wherever a `DomainID` is required.

#### 2.1.2. `AcceptedCredentials`

This is an array of `Credentials` objects. The maximum length of this array is 10.

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`Issuer`|✔️|`string`|`AccountID`|The issuer of the credential.|
|`CredentialType`| |`string`|`Blob`|A value to identify the type of credential from the issuer.|

#### 2.1.3. `AcceptedTokens`

This is an array of `Issue` objects, which represent issued currencies on the XRPL. The maximum length of this array is 10. For each element, either an issued currency or an MPT must be specified.

XRP is automatically included, and therefore will not be included in this array.

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`Token`| |`object`|`Issue`|The token.|
|`MPToken`| |`object`|`MPTIssue`|The MPToken.|

### 2.2. Account Deletion

The `PermissionedDomain` object is not a [deletion blocker](https://xrpl.org/docs/concepts/accounts/deleting-accounts/#requirements).

## 3. Transaction: `PermissionedDomainSet`

This transaction creates or modifies a `PermissionedDomain` object.

### 3.1. Fields

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`TransactionType`| ✔️|`string`|`UInt16`|The transaction type (`PermissionedDomainSet`).|
|`Account`| ✔️|`string`|`AccountID`|The account sending the transaction.|
|`DomainID`| |`string`|`Hash256`|The domain to modify. Must be included if modifying an existing domain.|
|`AcceptedCredentials`| |`array`|`STArray`|The credentials that are accepted by the domain. Ownership of one of these credentials automatically makes you a member of the domain. An empty array means deleting the field.|
|`AcceptedTokens`| |`array`|`STArray`|The tokens that are allowed by the domain. An empty array means deleting the field.|

### 3.2. Failure Conditions

* `Issuer` doesn't exist on one or more of the credentials in `AcceptedCredentials`.
* The resulting `PermissionedDomain` object doesn't have any accepted credentials or tokens.
* The `AcceptedTokens` or `AcceptedCredentials` arrays are too long.
* XRP is included in the `AcceptedTokens` array (since it's included by default).
* If `DomainID` is included:
	* That domain doesn't exist.
	* The account isn't the domain owner.

### 3.3. State Changes

* Creates or modifies a `PermissionedDomain` object.

## 4. Transaction: `PermissionedDomainDelete`

This transaction deletes a `PermissionedDomain` object.

### 4.1. Fields

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`TransactionType`| ✔️|`string`|`UInt16`|The transaction type (`PermissionedDomainDelete`).|
|`Account`| ✔️|`string`|`AccountID`|The account sending the transaction.|
|`DomainID`| |`string`|`Hash256`|The domain to delete.|

### 4.2. Failure Conditions

* The domain specified in `DomainID` doesn't exist.
* The account isn't the owner of the domain.

### 4.3. State Changes

* Deletes a `PermissionedDomain` object.

## 5. On-Ledger Object: `Offer`

The `Offer` object tracks an offer placed on the CLOB DEX. This object type already exists on the XRPL, but is being extended as a part of this spec to also support permissioned DEX domains.

### 5.1. Fields

<details>
<summary>

As a reference, [here](https://xrpl.org/docs/references/protocol/ledger-data/ledger-entry-types/offer) are the existing fields for the `Offer` object.
</summary>

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`Account`|✔️|`string`|`AccountID`|The address of the account that owns this offer.|
|`BookDirectory`|✔️|`string`|`Hash256`|The ID of the offer directory that links to this offer.|
|`BookNode`|✔️|`string`|`UInt64`|A hint indicating which page of the offer directory links to this entry, in case the directory consists of multiple pages.|
|`Expiration`| |`number`|`UInt32`|Indicates the time after which this offer is considered unfunded.|
|`LedgerEntryType`|✔️|`string`|`UInt16`|The value `0x006F`, mapped to the string `Offer`, indicates that this is an offer entry.|
|`OwnerNode`|✔️|`string`|`UInt64`|A hint indicating which page of the owner directory links to this entry, in case the directory consists of multiple pages.|
|`PreviousTxnID`|✔️|`string`|`Hash256`|The identifying hash of the transaction that most recently modified this entry.|
|`PreviousTxnLgrSeq`|✔️|`number`|`UInt32`|The ledger index that contains the transaction that most recently modified this object.|
|`Sequence`|✔️|`number`|`UInt32`|The `Sequence` value of the `OfferCreate` transaction that created this offer. Used in combination with the `Account` to identify this offer.|
|`TakerPays`|✔️|`object`|`Amount`|The remaining amount and type of currency requested by the offer creator.|
|`TakerGets`|✔️|`object`|`Amount`|The remaining amount and type of currency being provided by the offer creator.|
</details>

We propose these additions:

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`DomainID`| |`string`|`Hash256`|The domain that the offer must be a part of.|

#### 5.1.1. `DomainID`

A permissioned offer has a `DomainID` field included.

An open offer does not need to include a `DomainID` field.

### 5.2. Offer Invalidity

An offer on the orderbook can become invalid if the domain specified by the `DomainID` field is deleted, and will no longer be filled. It will be automatically deleted when crossed, like an unfunded offer.

The same will be true if the credential that allows the offer's owner to be a member of the domain expires or is deleted.

## 6. Transaction: `OfferCreate`

The `OfferCreate` transaction creates an offer on the CLOB DEX. This transaction type already exists on the XRPL, but is being extended as a part of this spec to also support permissioned DEX domains.

### 6.1. Fields

<details>
<summary>

As a reference, [here](https://xrpl.org/docs/references/protocol/transactions/types/offercreate) are the existing fields for the `OfferCreate` transaction.
</summary>

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`Expiration`| |`number`|`UInt32`|Time after which the offer is no longer active, in seconds since the Ripple Epoch.|
|`OfferSequence`| |`number`|`UInt32`|An offer to delete first, specified in the same way as `OfferCancel`.|
|`TakerGets`|✔️|`string`|`Amount`|The amount and type of currency being sold.|
|`TakerPays`|✔️|`string`|`Amount`|The amount and type of currency being bought.|
</details>

We propose these additions:

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`DomainID`| |`string`|`Hash256`|The domain that the offer must be a part of.|

### 6.2. Failure Conditions

The existing set of failure conditions for `OfferCreate` will continue to exist.

There will also be the following in addition, if the `DomainID` field is included:
* The domain doesn't exist.
* The offer is not a valid domain offer.

### 6.3. State Changes

The existing set of state changes for `OfferCreate` will continue to exist.

If the `DomainID` is included in the `OfferCreate` transaction, whenever an offer is crossed:
* If the crossed offer doesn't include a `DomainID`, the offer will be checked to see if it is a valid domain offer. If it does, it will fill that offer as per the rules of the `OfferCreate`'s parameters. If it doesn't, the offer will be skipped.
* If the crossed offer does include a `DomainID`, it must match the `DomainID` in the transaction (even if each would be a valid domain offer in the other domain).
* If there is an offer placed on the ledger, it will include the `DomainID`.

If the `DomainID` is not included in the `OfferCreate` transaction, whenever an offer with a `DomainID` is crossed:
* The offer outlined in the `OfferCreate` transaction will be checked to see if it is a valid domain offer. If it does, it will fill that offer as per the rules of the `OfferCreate`'s parameters. If it doesn't, the offer will be skipped.
	* If the on-ledger offer is no longer a valid domain offer (i.e. the credential expired, or was removed from the domain), the offer will be deleted (just as it would be if it were unfunded).

## 7. Transaction: `Payment`

A `Payment` transaction represents a transfer of value from one account to another, and can involve currency conversions and crossing the orderbook. This transaction type already exists on the XRPL, but is being extended as a part of this spec to also support permissioned DEX domains.

### 7.1. Fields

<details>
<summary>

As a reference, [here](https://xrpl.org/docs/references/protocol/transactions/types/payment) are the existing fields for the `Payment` transaction.
</summary>

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`Amount`|✔️|`string`|`Amount`|The maximum amount of currency to deliver. For non-XRP amounts, the nested field names MUST be lower-case. If the `tfPartialPayment` flag is set, deliver _up to_ this amount instead.|
|`DeliverMin`||`string`|`Amount`|The minimum amount of destination currency this transaction should deliver. Only valid if this is a partial payment. For non-XRP amounts, the nested field names are lower-case.|
|`Destination`|✔️|`string`|`AccountID`|The unique address of the account receiving the payment.|
|`DestinationTag`||`number`|`UInt32`|An arbitrary tag that identifies the reason for the payment to the destination, or a hosted recipient to pay.|
|`InvoiceID`||`string`|`Hash256`|An arbitrary 256-bit hash representing a specific reason or identifier for this payment.|
|`Paths`||`array`|`PathSet`|Array of payment paths to be used for this transaction. Must be omitted for XRP-to-XRP transactions.|
|`SendMax` ||`string` or `object`|`Amount`|Highest amount of source currency this transaction is allowed to cost, including transfer fees, exchange rates, and [slippage](http://en.wikipedia.org/wiki/Slippage_%28finance%29). Does not include the XRP destroyed as a cost for submitting the transaction. For non-XRP amounts, the nested field names MUST be lower-case. Must be supplied for cross-currency/cross-issue payments. Must be omitted for XRP-to-XRP payments.|.|
</details>

We propose these additions:

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`DomainID`| |`string`|`Hash256`|The domain that the offer must be a part of.|

#### 7.1.1. `DomainID`

The `DomainID` can only be included if the payment is a cross-currency payment (i.e. if the payment is going to interact with the DEX). It should only be included if the payment is permissioned.

### 7.2. Failure Conditions

The existing set of failure conditions for `Payment` will continue to exist.

There will also be the following in addition, if the `DomainID` is included:
* The payment isn't a cross-currency payment.
* The domain doesn't exist.
* The `Account` is not a domain member.
* The currencies used in `Amount`, `SendMax`, and `DeliverMin` are not permitted as a part of the domain's rules.
* The paths do not satisfy the domain's rules.

### 7.3. State Changes

The existing set of state changes for `Payment` will continue to exist.

## 8. RPC: `book_offers`

The [`book_offers` RPC method](https://xrpl.org/book_offers.html) already exists on the XRPL. This proposal suggests some modifications to also support permissioned DEX domains.

### 8.1. Request Fields

As a reference, here are the fields that `book_offers` currently accepts:

| Field Name | Required? | JSON Type | Description |
|------------|-----------|-----------|-------------|
|`taker_gets`|✔️|`object`|The asset the account taking the Offer would receive, as a currency without an amount.|
|`taker_pays`|✔️|`object`|The asset the account taking the Offer would pay, as a currency without an amount.|
|`ledger_hash`| |`string`|A 20-byte hex string for the ledger version to use.|
|`ledger_index`| |`number` or `string`|The ledger index of the ledger to use, or a shortcut string to choose a ledger automatically.|
|`limit`| |`number`|The maximum number of Offers to return. The response may include fewer results.|
|`taker`| |`string`|The address of an account to use as a perspective. The response includes this account's offers even if they are unfunded. (You can use this to see what offers are above or below yours in the order book.)|

This proposal puts forward the following addition:

| Field Name | Required? | JSON Type | Description |
|------------|-----------|-----------|-------------|
|`domain`| |`string`|The object ID of a `PermissionedDomain` object. If this field is included, then the offers will be filtered to only show the valid domain offers for that domain.|

### 8.2. Response Fields

This proposal does not suggest any changes to the response fields. As a reference, here are the fields that `book_offers` currently returns:

| Field Name | Always Present? | JSON Type | Description |
|------------|-----------------|-----------|-------------|
|`ledger_current_index`| |`number` or `string`|The ledger index of the current in-progress ledger version, which was used to retrieve this information.|
|`ledger_index`| |`number` or `string`|The ledger index of the ledger version that was used when retrieving this data, as requested.|
|`ledger_hash`| |`string`|The identifying hash of the ledger version that was used when retrieving this data, as requested.|
|`offers`|✔️|`array`|Array of offer objects, each of which has the fields of an Offer object.|


## 9. RPC: `path_find`

The [`path_find` RPC method](https://xrpl.org/path_find.html) already exists on the XRPL. This proposal suggests some modifications to also support permissioned DEX domains.

Only the `create` subcommand will be affected.

### 9.1. Request Fields

| Field Name | Required? | JSON Type | Description |
|------------|-----------|-----------|-------------|
|`subcommand`|✔️|`string`|Use `"create"` to send the create sub-command|
|`source_account`|✔️|`string`|The address of the account to find a path from. (In other words, the account that would be sending a payment.)|
|`destination_account`|✔️|`string`|The address of the account to find a path to. (In other words, the account that would receive a payment.)|
|`destination_amount` |✔️|`string` or `object`|The currency amount that the destination account would receive in a transaction.|
|`send_max`| |`string` or `object`| The currency amount that would be spent in the transaction.|
|`paths`| |`array`|Array of arrays of objects, representing payment paths to check. You can use this to keep updated on changes to particular paths you already know about, or to check the overall cost to make a payment along a certain path.|

This proposal puts forward the following addition:

| Field Name | Required? | JSON Type | Description |
|------------|-----------|-----------|-------------|
|`domain`| |`string`|The object ID of a `PermissionedDomain` object. If this field is included, then the paths will be filtered to only show the valid paths for that domain.|

### 9.2. Response Fields

| Field Name | Always Present? | JSON Type | Description |
|------------|-----------------|-----------|------------|
|`alternatives`|✔️|`array`|An array of objects with suggested paths to take, as described below. If empty, then no paths were found connecting the source and destination accounts.|
|`destination_account`|✔️|`string`|The address of the account that would receive a transaction.|
|`destination_amount`|✔️|`string` or `object`|The currency amount that the destination would receive in a transaction.|
|`source_account`|✔️|`string`|The address that would send a transaction.|
|`full_reply`|✔️|`boolean`|If `false`, this is the result of an incomplete search. A later reply may have a better path. If `true`, then this is the best path found. (It is still theoretically possible that a better path could exist, but `rippled` won't find it.) Until you close the pathfinding request, `rippled` continues to send updates each time a new ledger closes.|

## 10. RPC: `ripple_path_find`

The [`ripple_path_find` RPC method](https://xrpl.org/ripple_path_find.html) already exists on the XRPL. This proposal suggests some modifications to also support permissioned DEX domains.

### 10.1. Request Fields

| Field Name | Required? | JSON Type | Description |
|------------|-----------|-----------|-------------|
|`source_account`|✔️|`string`|The address of the account that would send funds in a transaction|
|`destination_account`|✔️|`string`|The address of the account that would receive funds in a transaction|
|`destination_amount`|✔️|`string` or `object`|The currency amount that the destination account would receive in a transaction.|
|`send_max`| |`string` or `object`|The currency amount that would be spent in the transaction. Cannot be used with `source_currencies`.|
|`source_currencies`| |`array`|An array of currencies that the source account might want to spend. Each entry in the array should be a JSON object with a mandatory `currency` field and optional `issuer` field, like how currency amounts are specified.|
|`ledger_hash`| |`string`|A 20-byte hex string for the ledger version to use. |
|`ledger_index`| |`string` or `number`|The ledger index of the ledger to use, or a shortcut string to choose a ledger automatically.|

This proposal puts forward the following addition:

| Field Name | Required? | JSON Type | Description |
|------------|-----------|-----------|-------------|
|`domain`| |`string`|The object ID of a `PermissionedDomain` object. If this field is included, then the paths will be filtered to only show the valid paths for that domain.|

### 10.2. Response Fields

| Field Name | Always Present? | JSON Type | Description |
|------------|-----------------|-----------|------------|
|`alternatives`|✔️|`array`|An array of objects with possible paths to take, as described below. If empty, then there are no paths connecting the source and destination accounts.|
|`destination_account`|✔️|`string`|The address of the account that would receive a payment transaction.|
|`destination_currencies`|✔️|`array`|Array of strings representing the currencies that the destination accepts.|

## 11. Examples

TODO: add example transactions for the example flows laid out in 1.3

## 12. Invariants

* You cannot have a domain with no rules.
* No offer with a `DomainID` field will be filled by an invalid domain offer.
	* NOTE: This can't be done in the standard `InvariantChecks`, but could theoretically be embedded in the payment engine somewhere.

## 13. Security

* You have to trust the issuers of the credentials.
* You have to trust the domain creator. You can be your own domain creator, though.
* Potential performance concerns

## n+1. Open Questions

* Does a permissioned offer need to be able to be part of multiple domains? If so, the `DomainID` field should be replaced with a `DomainIDs` field (to accommodate a list)
* Should there be a flag on a domain to make it immutable? And/or undeleteable?
* Does the domain owner need to have trustlines for the tokens/hold the credentials?
* Does the domain owner need to be able to freeze/clawback tokens (from the PRD) or delete credentials?
	* If so, how? There's no real "domain membership" - if you hold one of the domain credentials you're a "domain member" - so it doesn't seem fair that a domain owner can freeze/clawback your tokens.
* Instead of having a single "Domain" object that stores all the rules, should the rules be each split out into their own object?
	* Would remove the number of rules restriction, but then each rule would cost one reserve.
	* Would make it easier to support black/whitelisting though (a la DepositAuth).
	* Update: I don't think this works because you need a list to iterate through.
* What other rule types might we want? (Not necessarily now, but also in the future)

# Appendix

## Appendix A: FAQ

### A.1: How are AMMs handled?

AMMs are not explicitly supported within permissioned DEXes in this proposal. They can participate in domains that are only token-gated (assuming the AMM's tokens are both part of the approved list), but not those that are credential-gated (since they cannot receive credentials). They could be added to permissioned DEXes in a future proposal.

### A.2: How will performance/TPS be affected from all these additional checks?

Performance tests will need to be conducted once the implementation is done. The spec may be modified if such tests show serious performance reduction.

### A.3: Why is the name `PermissionedDomain` so long? Why not shorten it to just `Domain`?

The term `Domain` is already used in the [account settings](https://xrpl.org/docs/references/protocol/transactions/types/accountset/#domain), so it would be confusing to use just the term `Domain` for this.

### A.4: Why do you need a domain? Why not just indicate what credentials you accept on the offer itself?



### A.5: Can a domain owner also be a credential issuer or token issuer?

Yes.

### A.6: Can I have a domain where XRP is _not_ an accepted token?

No, XRP must always be an accepted token, since transaction fees are paid in XRP and autobridging may also happen via XRP.

### A.7: Will an account with Deposit Authorization enabled be forced to use Permissioned DEXes? 

No.

## Appendix B: Alternate Designs

Maybe mention the platonic ideal design here
