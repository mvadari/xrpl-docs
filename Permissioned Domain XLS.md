<pre>
Title:       <b>Permissioned Domains</b>
Revision:    <b>1</b> (2024-08-20)

Author:      <a href="mailto:mvadari@ripple.com">Mayukha Vadari</a>

Affiliation: <a href="https://ripple.com">Ripple</a>
</pre>

# Permissioned Domains

## Abstract

This proposal introduces the concept of permissioned domains. Permissioned domains enable the creation of controlled environments within a broader system where specific rules and restrictions can be applied to user interactions and asset flow. This approach aims to bridge the gap between the transparency and security benefits of decentralized blockchain technology and the regulatory requirements of traditional financial institutions.

## 1. Overview

This proposal builds on top of [XLS-70d](https://github.com/XRPLF/XRPL-Standards/discussions/202), as credentials are needed for permissioning.

We propose:
* Creating a `PermissionedDomain` ledger object.
* Creating a `PermissionedDomainSet` transaction.
* Creating a `PermissionedDomainDelete` transaction.

This feature will require an amendment. Since this feature doesn't bring any functionality to the XRPL by itself, it will be gated by other amendments that use it instead.

### 1.1. Terminology

* **Domain**: A collection of rules indicating what accounts and trades may be a part of it. This spec includes credential-gating and token-gating, but more options could be added in the future.
* **Domain Rules**: The set of rules that govern a domain, i.e. the credentials and tokens it accepts.
* **Domain Owner**: The account that created a domain, and is the only one that can modify its rules or delete it.
* **Domain Member**: An account that satisfies the rules of the domain (i.e. has one of the credentials that are accepted by the domain). There is no explicit joining step; as long as the account has a valid credential, it is a member.

## 2. On-Ledger Object: `PermissionedDomain`

This object represents a permissioned domain.

### 2.1. Fields

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`LedgerIndex`|✔️|`string`|`Hash256`|The unique ID of the ledger object.|
|`LedgerEntryType`|✔️|`string`|`UInt16`|The ledger object's type (`PermissionedDomain`).|
|`Owner`|✔️|`string`|`AccountID`|The account that controls the settings of the domain.|
|`Sequence`|✔️|`number`|`UInt32`|The `Sequence` value of the `PermissionedDomainSet` transaction that created this domain. Used in combination with the `Account` to identify this domain.|
|`AcceptedCredentials`| |`array`|`STArray`|The credentials that are accepted by the domain. Ownership of one of these credentials automatically makes you a member of the domain.|
|`AcceptedTokens`| |`array`|`STArray`|The tokens that are allowed by the domain.|

#### 2.1.1. `LedgerIndex`

The ID of this object will be a hash that incorporates the `Owner` and `Sequence` fields, combined with a unique space key for `PermissionedDomain` objects, which will be defined during implementation.

This value will be used wherever a `DomainID` is required.

#### 2.1.2. `AcceptedCredentials`

This is an array of inner objects referencing a type of credential. The maximum length of this array is 10.

The array will be sorted by `Issuer`, so that searching it for a match is more performant.

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`Issuer`|✔️|`string`|`AccountID`|The issuer of the credential.|
|`CredentialType`|✔️|`string`|`Blob`|A value to identify the type of credential from the issuer.|

#### 2.1.3. `AcceptedTokens`

This is an array of `Issue` objects, which represent issued currencies on the XRPL. The maximum length of this array is 10. For each element, either an issued currency or an MPT must be specified.

XRP is automatically included, and therefore will not be included in this array.

The array will be sorted by `issuer`, so that searching it for a match is more performant.

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`Token`| |`object`|`Issue`|The token.|
|`MPToken`| |`object`|`MPTIssue`|The MPToken.|

### 2.2. Account Deletion

The `PermissionedDomain` object is a [deletion blocker](https://xrpl.org/docs/concepts/accounts/deleting-accounts/#requirements).

## 3. Transaction: `PermissionedDomainSet`

This transaction creates or modifies a `PermissionedDomain` object.

### 3.1. Fields

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`TransactionType`|✔️|`string`|`UInt16`|The transaction type (`PermissionedDomainSet`).|
|`Account`|✔️|`string`|`AccountID`|The account sending the transaction.|
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

If the transaction is successful:
* It creates or modifies a `PermissionedDomain` object.

## 4. Transaction: `PermissionedDomainDelete`

This transaction deletes a `PermissionedDomain` object.

### 4.1. Fields

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`TransactionType`|✔️|`string`|`UInt16`|The transaction type (`PermissionedDomainDelete`).|
|`Account`|✔️|`string`|`AccountID`|The account sending the transaction.|
|`DomainID`| |`string`|`Hash256`|The domain to delete.|

### 4.2. Failure Conditions

* The domain specified in `DomainID` doesn't exist.
* The account isn't the owner of the domain.

### 4.3. State Changes

If the transaction is successful:
* It deletes a `PermissionedDomain` object.

## 5. Examples

TODO: add an example domain object

## 6. Invariants

* You cannot have a domain with no rules.
* The `AcceptedCredentials` and `AcceptedTokens` arrays must both have length between 1 and 10.

## 7. Security

* You have to trust the issuers of the credentials.
* You have to trust the domain creator. You can be your own domain creator, though.

## n+1. Open Questions

* Should there be a flag on a domain to make it immutable? And/or undeleteable?
* Does the domain owner need to have trustlines for the tokens/hold the credentials?
* Does the domain owner need to be able to freeze/clawback tokens (from the PRD) or delete credentials?
	* If so, how? There's no real "domain membership" - if you hold one of the domain credentials you're a "domain member" - so it doesn't seem fair that a domain owner can freeze/clawback your tokens.
	* Update: No.
* Instead of having a single "Domain" object that stores all the rules, should the rules be each split out into their own object?
	* Would remove the number of rules restriction, but then each rule would cost one reserve.
	* Would make it easier to support black/whitelisting though (a la DepositAuth).
	* Each account could probably only have one domain, then.
	* Update: I don't think this works because you need a list to iterate through.
* What other rule types might we want? (Not necessarily now, but also in the future)
* Should there be a "domain membership" object? It would enable blacklisting etc.

# Appendix

## Appendix A: FAQ

### A.1: Why is the name `PermissionedDomain` so long? Why not shorten it to just `Domain`?

The term `Domain` is already used in the [account settings](https://xrpl.org/docs/references/protocol/transactions/types/accountset/#domain), so it would be confusing to use just the term `Domain` for this.

### A.2: Can a domain owner also be a credential issuer or token issuer?

Yes.

### A.3: Can I have a domain where XRP is _not_ an accepted token?

No, XRP must always be an accepted token, since transaction fees are paid in XRP and autobridging may also happen via XRP.

### A.4: Can other rule types be added to domains?

Yes, if there is a need.

### A.5: Can I AND credentials together?

No, because it is difficult to make a clean design for that capability.

