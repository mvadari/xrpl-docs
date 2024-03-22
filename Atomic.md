<pre>
Title:       <b>Atomic/Batch Transactions</b>
Revision:    <b>3</b> (2024-03-19)

Author:      <a href="mailto:mvadari@ripple.com">Mayukha Vadari</a>

Affiliation: <a href="https://ripple.com">Ripple</a>
</pre>

# Atomic/Batch Transactions

## Abstract

The XRP Ledger has a robust set of built-in features, enabling fast and efficient transactions without the need for complex smart contracts on every step. However, a key limitation exists: multiple transactions cannot be executed atomically. This means that if a complex operation requires several transactions, a failure in one can leave the system in an incomplete or unexpected state. Imagine building a house: you wouldn't want to lay the foundation and build the walls, and then discover you can't afford the roof, leaving you with an unfinished and unusable structure.

This document proposes a design for atomic transactions, a functionality that allows multiple transactions to be packaged together and executed as a single unit. It's like laying the foundation, building the walls, and raising the roof all in one single secure step, leveraging the existing strengths of the XRP Ledger. If you're unable to afford the roof, you won't even bother laying the foundation.

This eliminates the risk of partial completion and unexpected outcomes, fostering a more reliable and predictable user experience for complex operations. By introducing atomic transactions, developers gain the ability to design innovative features and applications that were previously hindered by the lack of native smart contracts for conditional workflows. This empowers them to harness the full potential of the XRP Ledger's built-in features while ensuring robust execution of complex processes.

Some use-cases that may be enabled by atomic transactions include:
* All or nothing: Mint an NFT and create an offer for it in one transaction. If the offer creation fails, the NFT mint is reverted as well.
* Trying out a few offers: Submit multiple offers with different amounts of slippage, but only one will succeed.
* Platform fees: Package platform fees within the transaction itself, simplifying the process.
* Atomic swaps (multi-account): Trustless token/NFT swaps between multiple accounts.
* Withdrawing accounts (multi-account): Attempt a withdrawal from your checking account, and if that fails, withdraw from your savings account instead.

## 1. Overview

This spec proposes one new transaction: `Atomic`. It will not require any new ledger objects, nor modifications to existing ledger objects. It will require an amendment, tentatively titled `featureAtomic`.

The rough idea of this design is that users can include "sub-transactions" inside `Atomic`, and these transactions are processed atomically. The design also supports transactions from different accounts in the same `Atomic` wrapper transaction.

### 1.1. Terminology
* **Inner transaction**: the sub-transactions included in the `Atomic` transaction, that are executed atomically.
* **Outer transaction**: the wrapper `Atomic` transaction itself.

## 2. Transaction: `Atomic`

|FieldName | Required? | JSON Type | Internal Type |
|:---------|:-----------|:---------------|:------------|
|`TransactionType`|✔️|`string`|`UInt16`|
|`Account`|✔️|`string`|`STAccount`|
|`Fee`|✔️|`string`|`STAmount`|
|`AtomicityType`|✔️|`number`|`UInt8`|
|`RawTransactions`|!|`array`|`STArray`|
|`TxnIDs`|✔️|`array`|`Vector256`|
|`AtomicSigners`| |`array`|`STArray`|

<!--
```typescript
{
    TransactionType: "Atomic",
    Account: "r.....",
    AtomicityType: n,
    TxnIDs: [transaction hashes...]
    RawTransactions: [transaction blobs...], // not included in the signature or stored on ledger
    AtomicSigners: [ // only sign the list of transaction hashes and probably the atomicity type
      AtomicSigner: {
        Account: "r.....",
        Signature: "...."
      },
      AtomicSigner: {
        Account: "r.....",
        Signers: [...] // multisign
      },
      ...
    ],
    SigningPubKey: "....",
    TxnSignature: "...."
}
```
-->

### 2.1. `Fee`

$$(n+2)*base\textunderscore fee$$
(where `n` is the number of signatures included in the outer transaction)

In other words, the fee is twice the base fee (a total of 20 drops when there is no fee escalation). This is just the fee for processing the atomicity overhead; each inner transaction handles its own fees.
 
### 2.2. `AtomicityType`

This spec supports three types of atomicity: `ALL`, `ONLYONE`, and `BATCH` (each will have an integer value, and tooling can handle the translation between integer values and the string they represent). 

If the `ALL` atomicity type is used, then all transactions must succeed for any of them to succeed.

If the `ONLYONE` atomicity type is used, then the first transaction to succeed will be the only one to succeed; all other transactions either failed or were never tried. While this can sort of be done by submitting multiple transactions with the same sequence number, there is no guarantee that the transactions are processed in the same order they are sent.

If the `BATCH` atomicity type is used, then all transactions will be applied until the first failure.

### 2.3. `RawTransactions`

`RawTransactions` contains the list of transactions that will be applied. There can be up to 8 transactions included. These transactions can come from one account or multiple accounts.

Each inner transaction:
* **Must** include a sequence number
* **Must** include a fee
* **Must not** be signed (the global transaction is already signed by all relevant parties). They must instead have an empty string (`""`) in the `SigningPubKey` and `TxnSignature` fields.

A transaction will be considered a failure if it receives any result that is not `tesSUCCESS`.

**This field is not included in the validated transaction**, since all transactions are included separately as a part of the ledger.

### 2.4. `TxnIDs`

`TxnIDs` contains a list of the transaction hashes/IDs for all the transactions contained in `RawTransactions`. This is the only part of the inner transactions that is saved as a part of the ledger within the `Atomic` transaction, since the inner transactions themselves will be their own transactions on-ledger. The hashes in `TxnIDs` **must** be in the same order as the raw transactions in `RawTransactions`.

While this field seems complicated/confusing to work with, it can easily be abstracted away (e.g. as a part of autofilling) in tooling, and it's easy for `rippled` to check a hash doesn't match its corresponding transaction in `RawTransaction`.

### 2.5. `AtomicSigners`

This field operates similarly to [multisign](https://xrpl.org/docs/concepts/accounts/multi-signing/) on the XRPL. It is only needed if multiple accounts' transactions are included in the `Atomic` transaction; otherwise, the normal transaction signature provides the same security guarantees.

|FieldName | Required? | JSON Type | Internal Type |
|:---------|:-----------|:---------------|:------------|
|`Account`|✔️|`string`|`STAccount`|
|`SigningPubKey`| |`string`|`STBlob`|
|`Signature`| |`string`|`STBlob`|
|`Signers`| |`array`|`STArray`|

#### 2.5.1. `Account`

This is an account that has at least one inner transaction.

#### 2.5.2. `SigningPubKey` and `Signature`

These fields are included if the account is signing with a single signature (as opposed to multi-sign). They sign the `AtomicityType` and `TxnIDs` fields.

#### 2.5.3. `Signers`

This field is included if the account is signing with multi-sign (as opposed to a single signature). It operates equivalently to the [`Signers` field](https://xrpl.org/docs/references/protocol/transactions/common-fields/#signers-field) used in standard transaction multi-sign. This field holds the signatures for the `AtomicityType` and `TxnIDs` fields.

### 2.6. Metadata

The inner transactions will be committed separately to the ledger and will therefore have separate metadata.

For example, a ledger that only has one `Atomic` transaction containing 2 inner transactions would look like this:
```
[
  OuterTransaction,
  InnerTransaction1,
  InnerTransaction2
]
```

#### 2.6.1. Outer Transactions

Each outer transaction will only contain the metadata for its sequence and fee processing, not for the inner transaction processing.

There will also be a list of which transactions were actually processed, which is useful for the `ONLYONE` and `BATCH` atomicity types, since those may only process a subset of transactions.

TODO: include examples and names of `AtomicExecutions`

#### 2.6.2. Inner Transactions

Each inner transaction will contain the metadata for its own processing. Only the inner transactions that were actually committed to the ledger will be included. This makes it easier for legacy systems to still be able to process `Atomic` transactions as if they were normal.

There will also be a pointer back to the parent outer transaction (`parent_atomic`), for ease of development (similar to the `nftoken_id` field).

## 3. Examples

### 3.1. One Account

In this example, the user is creating an offer while trading on a DEX UI, and the second transaction is a platform fee.

#### 3.1.1. Sample Transaction

<details open>
<summary>

The inner transactions are not signed, and the `AtomicSigners` field is not needed on the outer transaction, since there is only one account involved.
</summary>

```typescript
{
  TransactionType: "Atomic",
  Account: "rUserBSM7T3b6nHX3Jjua62wgX9unH8s9b",
  AtomicityType: 0,
  TxnIDs: [
    "7EB435C800D7DC10EAB2ADFDE02EE5667C0A63AA467F26F90FD4CBCD6903E15E",
    "EAE6B33078075A7BA958434691B896CCA4F532D618438DE6DDC7E3FB7A4A0AAB"
  ],
  RawTransactions: [
    {
      RawTransaction: {
        TransactionType: "OfferCreate",
        Account: "rUserBSM7T3b6nHX3Jjua62wgX9unH8s9b",
        TakerGets: "6000000",
        TakerPays: {
          currency: "GKO",
          issuer: "ruazs5h1qEsqpke88pcqnaseXdm6od2xc",
          value: "2"
        },
        Sequence: 4,
        Fee: "10",
        SigningPubKey: "",
        TxnSignature: ""
      }
    },
    {
      RawTransaction: {
        TransactionType: "Payment",
        Account: "rUserBSM7T3b6nHX3Jjua62wgX9unH8s9b",
        Destination: "rDEXfrontEnd23E44wKL3S6dj9FaXv",
        Amount: "1000",
        Sequence: 5,
        Fee: "10",
        SigningPubKey: "",
        TxnSignature: ""
      }
    }
  ],
  Sequence: 3,
  Fee: "20",
  SigningPubKey: "022D40673B44C82DEE1DDB8B9BB53DCCE4F97B27404DB850F068DD91D685E337EA",
  TxnSignature: "3045022100EC5D367FAE2B461679AD446FBBE7BA260506579AF4ED5EFC3EC25F4DD1885B38022018C2327DB281743B12553C7A6DC0E45B07D3FC6983F261D7BCB474D89A0EC5B8"
}
```
</details>

#### 3.1.2. Sample Ledger

<details open>
<summary>
This example shows what the ledger will look like after the transaction is confirmed.

Note that the inner transactions are committed as normal transactions, and the `RawTransactions` field is not included in the validated version of the outer transaction.
</summary>

```typescript
[
  {
    TransactionType: "Atomic",
    Account: "rUserBSM7T3b6nHX3Jjua62wgX9unH8s9b",
    AtomicityType: 0,
    TxnIDs: [
      "7EB435C800D7DC10EAB2ADFDE02EE5667C0A63AA467F26F90FD4CBCD6903E15E",
      "EAE6B33078075A7BA958434691B896CCA4F532D618438DE6DDC7E3FB7A4A0AAB"
    ],
    Sequence: 3,
    Fee: "20",
    SigningPubKey: "022D40673B44C82DEE1DDB8B9BB53DCCE4F97B27404DB850F068DD91D685E337EA",
    TxnSignature: "3045022100EC5D367FAE2B461679AD446FBBE7BA260506579AF4ED5EFC3EC25F4DD1885B38022018C2327DB281743B12553C7A6DC0E45B07D3FC6983F261D7BCB474D89A0EC5B8"
  },
  {
    TransactionType: "OfferCreate",
    Account: "rUserBSM7T3b6nHX3Jjua62wgX9unH8s9b",
    TakerGets: "6000000",
    TakerPays: {
      currency: "GKO",
      issuer: "ruazs5h1qEsqpke88pcqnaseXdm6od2xc",
      value: "2"
    },
    Sequence: 4,
    Fee: "10",
    SigningPubKey: "",
    TxnSignature: ""
  },
  {
    TransactionType: "Payment",
    Account: "rUserBSM7T3b6nHX3Jjua62wgX9unH8s9b",
    Destination: "rDEXfrontEnd23E44wKL3S6dj9FaXv",
    Amount: "1000",
    Sequence: 5,
    Fee: "10",
    SigningPubKey: "",
    TxnSignature: ""
  }
]
```
</details>

### 3.2. Multiple Accounts

In this example, two users are atomically swapping their tokens, XRP for GKO.

#### 3.2.1. Sample Transaction

<details open>
<summary>

The inner transactions are still not signed, but the `AtomicSigners` field is needed on the outer transaction, since there are two accounts' inner transactions in this `Atomic` transaction.
</summary>

```typescript
{
  TransactionType: "Atomic",
  Account: "rUser1fcu9RJa5W1ncAuEgLJF2oJC6",
  AtomicityType: 0,
  TxnIDs: [
    "A2986564A970E2B206DC8CA22F54BB8D73585527864A4484A5B0C577B6F13C95",
    "0C4316F7E7D909E11BB7DBE0EB897788835519E9950AE8E32F5182468361FE7E"
  ],
  RawTransactions: [
    {
      RawTransaction: {
        TransactionType: "Payment",
        Account: "rUser1fcu9RJa5W1ncAuEgLJF2oJC6",
        Destination: "rUser2fDds782Bd6eK15RDnGMtxf7m",
        Amount: "6000000",
        Sequence: 5,
        Fee: "10",
        SigningPubKey: "",
        TxnSignature: ""
      }
    },
    {
      RawTransaction: {
        TransactionType: "Payment",
        Account: "rUser2fDds782Bd6eK15RDnGMtxf7m",
        Destination: "rUser1fcu9RJa5W1ncAuEgLJF2oJC6",
        Amount: {
          currency: "GKO",
          issuer: "ruazs5h1qEsqpke88pcqnaseXdm6od2xc",
          value: "2"
        },
        Sequence: 5,
        Fee: "10",
        SigningPubKey: "",
        TxnSignature: ""
      }
    }
  ],
  AtomicSigners: [
    {
      AtomicSigner: {
        Account: "rUser1fcu9RJa5W1ncAuEgLJF2oJC6",
        SigningPubKey: "03072BBE5F93D4906FC31A690A2C269F2B9A56D60DA9C2C6C0D88FB51B644C6F94",
        Signature: "304502210083DF12FA60E2E743643889195DC42C10F62F0DE0A362330C32BBEC4D3881EECD022010579A01E052C4E587E70E5601D2F3846984DB9B16B9EBA05BAD7B51F912B899"
      }
    },
    {
      AtomicSigner: {
        Account: "rUser2fDds782Bd6eK15RDnGMtxf7m",
        SigningPubKey: "03C6AE25CD44323D52D28D7DE95598E6ABF953EECC9ABF767F13C21D421C034FAB",
        Signature: "304502210083DF12FA60E2E743643889195DC42C10F62F0DE0A362330C32BBEC4D3881EECD022010579A01E052C4E587E70E5601D2F3846984DB9B16B9EBA05BAD7B51F912B899"
      }
    },
  ],
  Sequence: 4,
  Fee: "40",
  SigningPubKey: "03072BBE5F93D4906FC31A690A2C269F2B9A56D60DA9C2C6C0D88FB51B644C6F94",
  TxnSignature: "30440220702ABC11419AD4940969CC32EB4D1BFDBFCA651F064F30D6E1646D74FBFC493902204E5B451B447B0F69904127F04FE71634BD825A8970B9467871DA89EEC4B021F8"
}
```
</details>

#### 3.2.2. Sample Ledger

<details open>
<summary>
This example shows what the ledger will look like after the transaction is confirmed.

Note that the inner transactions are committed as normal transactions, and the `RawTransactions` field is not included in the validated version of the outer transaction.
</summary>

```typescript
[
  {
    TransactionType: "Atomic",
    Account: "rUser1fcu9RJa5W1ncAuEgLJF2oJC6",
    AtomicityType: 0,
    TxnIDs: [
      "A2986564A970E2B206DC8CA22F54BB8D73585527864A4484A5B0C577B6F13C95",
      "0C4316F7E7D909E11BB7DBE0EB897788835519E9950AE8E32F5182468361FE7E"
    ],
    AtomicSigners: [
      {
        AtomicSigner: {
          Account: "rUser1fcu9RJa5W1ncAuEgLJF2oJC6",
          SigningPubKey: "03072BBE5F93D4906FC31A690A2C269F2B9A56D60DA9C2C6C0D88FB51B644C6F94",
          Signature: "304502210083DF12FA60E2E743643889195DC42C10F62F0DE0A362330C32BBEC4D3881EECD022010579A01E052C4E587E70E5601D2F3846984DB9B16B9EBA05BAD7B51F912B899"
        }
      },
      {
        AtomicSigner: {
          Account: "rUser2fDds782Bd6eK15RDnGMtxf7m",
          SigningPubKey: "03C6AE25CD44323D52D28D7DE95598E6ABF953EECC9ABF767F13C21D421C034FAB",
          Signature: "304502210083DF12FA60E2E743643889195DC42C10F62F0DE0A362330C32BBEC4D3881EECD022010579A01E052C4E587E70E5601D2F3846984DB9B16B9EBA05BAD7B51F912B899"
        }
      },
    ],
    Sequence: 4,
    Fee: "40",
    SigningPubKey: "03072BBE5F93D4906FC31A690A2C269F2B9A56D60DA9C2C6C0D88FB51B644C6F94",
    TxnSignature: "30440220702ABC11419AD4940969CC32EB4D1BFDBFCA651F064F30D6E1646D74FBFC493902204E5B451B447B0F69904127F04FE71634BD825A8970B9467871DA89EEC4B021F8"
  },
  {
    TransactionType: "Payment",
    Account: "rUser1fcu9RJa5W1ncAuEgLJF2oJC6",
    Destination: "rUser2fDds782Bd6eK15RDnGMtxf7m",
    Amount: "6000000",
    Sequence: 5,
    Fee: "10",
    SigningPubKey: "",
    TxnSignature: ""
  },
  {
    TransactionType: "Payment",
    Account: "rUser2fDds782Bd6eK15RDnGMtxf7m",
    Destination: "rUser1fcu9RJa5W1ncAuEgLJF2oJC6",
    Amount: {
      currency: "GKO",
      issuer: "ruazs5h1qEsqpke88pcqnaseXdm6od2xc",
      value: "2"
    },
    Sequence: 5,
    Fee: "10",
    SigningPubKey: "",
    TxnSignature: ""
  }
]
```
</details>

## 4. Security

### 4.1. Trust Assumptions

Regardless of how many accounts' transactions are included in an `Atomic` transaction, all accounts need to sign the collection of transactions.

#### 4.1.1. Single Account
In the single account case, this is obvious; the single account must approve all of the transactions it is submitting. No other accounts are involved, so this is a pretty straightforward case.

#### 4.1.2. Multi Account
The multi-account case is a bit more complicated and is best illustrated with an example. Let's say Alice and Bob are conducting a trustless atomic swap via a multi-account `Atomic`, with Alice providing 1000 XRP and Bob providing 1000 USD. Bob is going to submit the `Atomic` transaction, so Alice must provide her part of the swap to him.

If Alice provides a fully autofilled and signed transaction to Bob, Bob could submit Alice's transaction on the ledger without submitting his and receive the 1000 XRP without losing his 1000 USD. Therefore, the inner transactions must be unsigned. 

If Alice just signs her part of the `Atomic` transaction, Bob could modify his transaction to only provide 1 USD instead, thereby getting his 1000 XRP at a much cheaper rate. Therefore, the entire `Atomic` transaction (and all its inner transactions) must be signed by all parties.

# Appendix

## Appendix A: FAQ

### A.1: What if I want a more complex tree of transactions that are AND/XORed together? Can I nest `Atomic` transactions?

The original version of this spec supported nesting `Atomic` transactions. However, upon further analysis, that was deemed a bit too complicated for an initial version of `Atomic`, as most of the benefit from this new feature does not require nested transactions. Based on user and community need, this could be added as part of a V2.

### A.2: What if all of the transactions fail? Will a fee still be claimed?

Yes, just as they would if they were individually submitted. 

TODO: maybe only the batch fee would be claimed? Maybe that fee should be higher?

### A.3: Could there be an additional atomicity type for allowing all transactions to be processed regardless of whether they succeeded?

This is unnecessary, since that is equivalent to submitting the transactions normally.

### A.4: Why do I need to include sequence numbers in the transactions? How do I handle situations where one transaction might fail but the next might succeed?

Sequence numbers need to be included to avoid hash collisions with multiple identical transactions. Otherwise, e.g. two payment transactions for 1 XRP to the same account (for, say, a platform fee) would have the same contents and therefore the same hash.

To handle cases where sequence numbers may need to be "skipped", [tickets](https://xrpl.org/docs/concepts/accounts/tickets/) can be used.

### A.5: Would this feature enable greater frontrunning abilities?

That is definitely a concern. Ways to mitigate this are still being investigated. Some potential answers:
* Charge for more extensive path usage
* Have higher fees for atomic transactions
* Submit the atomic transactions at the end of the ledger

### A.6: What error is returned if all the transactions fail in an `OR`/`BATCH` transaction?

The answer to this question is still being investigated. Some potential answers:
* Return the first error encountered
* Return the last error encountered
* Return a general error, `temBATCH_FAILED`/`tecBATCH_FAILED` 
* Return a list of all the errors encountered in the metadata, for easier debugging

### A.7: Can another account sign/pay for the outer transaction if they don't have any of the inner transactions?

If there are multiple parties in the inner transactions, yes. Otherwise, no. This is because in a single party `Atomic` transaction, the inner transaction's signoff is provided by `AtomicSigners`.

### A.8: How does this work in conjunction with [XLS-49d](https://github.com/XRPLF/XRPL-Standards/discussions/144)? If I give a signer list powers over the `Atomic` transaction, can it effectively run all transaction types?

The answer to this question is still being investigated. Some potential answers:
* All signer lists should have access to this transaction but only for the transaction types they have powers over
* Only the global signer list can have access to this transaction

### A.9: Why not call this transaction `Batch`?

It has greater capabilities than just batching transactions. 

### A.10: What if I want some error code types to be allowed to proceed, just as `tesSUCCESS` would, in e.g. an `ALL` case?

This was deemed unnecessary. If you have a need for this, please provide example use-cases.

### A.11: What if I want the `Atomic` transaction signer to handle the fees for the inner transactions?

That is not supported in this version of the spec. This is due to the added complexity of passing info about who is paying the fee down the stack.
