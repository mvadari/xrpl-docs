<pre>
Title:       <b>Permissioned Domains</b>
Revision:    <b>1</b> (2024-09-12)

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

* **Domain**: A collection of rules indicating what accounts and transactions may be a part of it. This spec includes credential-gating and token-gating, but more options could be added in the future.
* **Domain Rules**: The set of rules that govern a domain, i.e. the credentials and tokens it accepts.
* **Domain Owner**: The account that created a domain, and is the only one that can modify its rules or delete it.
* **Domain Member**: An account that satisfies the rules of the domain (i.e. has one of the credentials that are accepted by the domain). There is no explicit joining step; as long as the account has a valid credential, it is a member.

## 2. On-Ledger Object: `PermissionedDomain`

This object represents a permissioned domain.

### 2.1. Fields

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`LedgerIndex`|✔️|`string`|`Hash256`|The unique ID of the ledger object.|
|`Flags`| ✔️|`number`|`UInt32`|Flag values associated with this object.|
|`LedgerEntryType`|✔️|`string`|`UInt16`|The ledger object's type (`PermissionedDomain`).|
|`Owner`|✔️|`string`|`AccountID`|The account that controls the settings of the domain.|
|`Sequence`|✔️|`number`|`UInt32`|The `Sequence` value of the `PermissionedDomainSet` transaction that created this domain. Used in combination with the `Account` to identify this domain.|
|`AcceptedCredentials`| |`array`|`STArray`|The credentials that are accepted by the domain. Ownership of one of these credentials automatically makes you a member of the domain.|
|`AcceptedTokens`| |`array`|`STArray`|The tokens that are allowed by the domain.|
|`PreviousTxnID`|✔️|`string`|`Hash256`|The identifying hash of the transaction that most recently modified this entry.|
|`PreviousTxnLgrSeq`|✔️|`number`|`UInt32`|The ledger index that contains the transaction that most recently modified this object.|

#### 2.1.1. `LedgerIndex`

The ID of this object will be a hash that incorporates the `Owner` and `Sequence` fields, combined with a unique space key for `PermissionedDomain` objects, which will be defined during implementation.

This value will be used wherever a `DomainID` is required.

#### 2.1.2. `AcceptedCredentials`

This is an array of inner objects referencing a type of credential. The maximum length of this array is 10.

The array will be sorted by `Issuer`, so that searching it for a match is more performant.

If this field is not included in the domain, then credentials do not need to be used when interacting with anything using the domain (i.e. there are no credential-based rules).

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`Issuer`|✔️|`string`|`AccountID`|The issuer of the credential.|
|`CredentialType`|✔️|`string`|`Blob`|A value to identify the type of credential from the issuer.|

#### 2.1.3. `AcceptedTokens`

This is an array of `Issue` objects, which represent issued currencies on the XRPL. The maximum length of this array is 10. For each element, either an issued currency or an MPT must be specified.

XRP is automatically included, and therefore will not be included in this array.

The array will be sorted by `issuer`, so that searching it for a match is more performant.

If this field is not included in the domain, then all tokens may be used when interacting with anything using the domain (i.e. there are no token-based rules).

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`Token`| |`object`|`Issue`|The token.|
|`MPToken`| |`object`|`MPTIssue`|The MPToken.|

#### 2.1.4. `Flags`

| Flag Name | Flag Value |
|-----------|------------|
|`lsfOnlyXRP`|`0x00010000`|

The `lsfOnlyXRP` flag represents whether the domain should only accept XRP (and no other tokens).

If the `lsfOnlyXRP` flag is enabled, then there must be no `AcceptedTokens` field.

To elaborate:
* If a token list is needed, the `AcceptedTokens` field should be included.
* If only XRP should be used, the `lsfOnlyXRP` flag should be included.
* If there are no token restrictions, neither should be included.

In other words, a `PermissionedDomain` object must have at least one of `lsfOnlyXRP`, `AcceptedTokens`, and `AcceptedCredentials`, but it also cannot have both `lsfOnlyXRP` and `AcceptedTokens`.

### 2.2. Account Deletion

The `PermissionedDomain` object is a [deletion blocker](https://xrpl.org/docs/concepts/accounts/deleting-accounts/#requirements).

## 3. Transaction: `PermissionedDomainSet`

This transaction creates or modifies a `PermissionedDomain` object.

### 3.1. Fields

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|-----------|-----------|---------------|-------------|
|`TransactionType`|✔️|`string`|`UInt16`|The transaction type (`PermissionedDomainSet`).|
|`Flags`| |`number`|`UInt32`|Set of bit-flags for this transaction.|
|`Account`|✔️|`string`|`AccountID`|The account sending the transaction.|
|`DomainID`| |`string`|`Hash256`|The domain to modify. Must be included if modifying an existing domain.|
|`AcceptedCredentials`| |`array`|`STArray`|The credentials that are accepted by the domain. Ownership of one of these credentials automatically makes you a member of the domain. An empty array means deleting the field.|
|`AcceptedTokens`| |`array`|`STArray`|The tokens that are allowed by the domain. An empty array means deleting the field.|

#### 3.1.1. `Flags`

| Flag Name | Flag Value |
|-----------|------------|
|`tfSetOnlyXRP`|`0x00010000`|
|`tfClearOnlyXRP`|`0x00020000`|

The `tfSetOnlyXRP` flag sets the `lsfOnlyXRP` flag on the `PermissionedDomain` object.

The `tfClearOnlyXRP` clears the `lsfOnlyXRP` flag on the `PermissionedDomain` object.

### 3.2. Failure Conditions

* `Issuer` doesn't exist on one or more of the credentials in `AcceptedCredentials`.
* The resulting `PermissionedDomain` object doesn't have any accepted credentials or tokens, or the `lsfOnlyXRP` flag.
* The `AcceptedTokens` or `AcceptedCredentials` arrays are too long.
* XRP is included in the `AcceptedTokens` array (since it's included by default).
* If `DomainID` is included:
	* That domain doesn't exist.
	* The account isn't the domain owner.
* The `tfSetOnlyXRP` flag is used and the `lsfOnlyXRP` flag is already set.
* The `tfClearOnlyXRP` flag is used and the `lsfOnlyXRP` flag is not set.

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

A sample domain object may look like this (ignoring common fields):

```typescript
{
  Owner: "rOWEN......",
  Sequence: 5,
  AcceptedCredentials: [
    {
      Credential: {
        Issuer: "rISABEL......",
        CredentialType: "123ABC"
      }
    }
  ],
  AcceptedTokens: [
    {
      Token: {
        currency: "USD",
        issuer: "rUSDISSUER......."
      }
    },
    {
      Token: {
        currency: "EUR",
        issuer: "rEURISSUER......."
      }
    }
  }
}
```

## 6. Invariants

* You cannot have a domain with no rules.
* The `AcceptedCredentials` and `AcceptedTokens` arrays must  have length between 1 and 10, if included.
* The `AcceptedTokens` field and `lsfOnlyXRP` flag cannot both be included.

## 7. Security

### 7.1. Issuer Trust

Relying on external issuers for credentials requires a degree of trust. If issuer credentials are compromised or forged, the system will be vulnerable.

### 7.2. Domain Creator Trust

While users can create their own domains, trust in the domain creator remains crucial. Malicious domain creators (or hackers) could potentially expose domain users to illegal liquidity.

# Appendix

## Appendix A: FAQ

### A.1: Why is the name `PermissionedDomain` so long? Why not shorten it to just `Domain`?

The term `Domain` is already used in the [account settings](https://xrpl.org/docs/references/protocol/transactions/types/accountset/#domain), so it would be confusing to use just the term `Domain` for this.

### A.2: Can a domain owner also be a credential issuer or token issuer?

Yes.

### A.3: Can I have a domain where XRP is _not_ an accepted token?

No, XRP must always be an accepted token, since transaction fees are paid in XRP and autobridging may also happen via XRP.

### A.4: Can other rule types be added to domains?

Yes, if there is a need. If you have any in mind, please mention them below.

### A.5: Can I AND credentials together?

No, because it is difficult to make a clean design for that capability.

### A.6: Does the domain owner have any special powers over the accepted tokens and credentials?

No, unless they are also the issuer of said token/credential.

### A.7: Does the domain owner need to have trustlines for the tokens/hold the credentials?

No.

### A.8: Why not have a ledger object for each domain rule, instead of having it all in one object? Then you wouldn't have any limitations on how many rules a domain could have.

This won't work.

The ledger needs to be able to iterate through the domain rules to ensure that all of them are being adhered to. If a domain owner has millions of objects (or millions of rules), then iterating through all of those objects becomes prohibitively expensive from a performance standpoint.

