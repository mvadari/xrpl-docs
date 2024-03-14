<pre>
Title:       <b>Atomic/Batch Transactions</b>
Revision:    <b>3</b> (2024-03-14)

Author:      <a href="mailto:mvadari@ripple.com">Mayukha Vadari</a>

Affiliation: <a href="https://ripple.com">Ripple</a>
</pre>

# Atomic/Batch Transactions

## Abstract

There is a need for transactions to be packaged together and executed atomically.

Potential use cases:
* Sending an `NFTokenMint` and `NFTokenCreateOffer` transaction in the same fell swoop (if the offer creation fails, so does the mint)
* Sending a few different offers that might fail and only wanting one to succeed
* Fees for platforms packaged with the actual transaction
* Atomic swaps (if multiple accounts are involved)

## 1. Overview

This spec proposes one new transaction: `Atomic`. It will not require any new ledger objects, nor modifications to existing ledger objects. It will require an amendment, tentatively titled `featureAtomic`.

### 1.1. Terminology
* **Inner transaction**: the sub-transactions included in the `Atomic` transaction, which are the same as the existing set of transactions.
* **Outer transaction**: the wrapper `Atomic` transaction itself.

## 2. Transaction: `Atomic`

|FieldName | Required? | JSON Type | Internal Type |
|:---------|:-----------|:---------------|:------------|
|`TransactionType`|✔️|`string`|`UInt16`|
|`Account`|✔️|`string`|`STAccount`|
|`Fee`|✔️|`string`|`STAmount`|
|`AtomicityType`|✔️|`number`|`UInt8`|
|`RawTransactions`|!|`array`|`STArray`|
|`Transactions`|✔️|`array`|`Vector256`|
|`BatchSigners`| |`array`|`STArray`|

```
{
    TransactionType: "Atomic",
    Account: "r.....",
    AtomicityType: n,
    Transactions: [transaction hashes...]
    RawTransactions: [transaction blobs...], // not included in the signature or stored on ledger
    BatchSigners: [ // only sign the list of transaction hashes and probably the atomicity type
      BatchSigner: {
        Account: "r.....",
        Signature: "...."
      },
      BatchSigner: {
        Account: "r.....",
        Signers: [...] // multisign
      },
      ...
    ],
    SigningPubKey: "....",
    TxnSignature: "...."
}
```

### 2.1. `Fee`

$$(n+2)*base\_fee$$
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
* **Must not** be signed (the global transaction is already signed by all relevant parties)

A transaction will be considered a failure if it receives any result that is not `tesSUCCESS`.

**This field is not included in the validated transaction**, since all transactions are included separately as a part of the ledger.

### 2.4. `Transactions`

`Transactions` contains a list of the transaction hashes for all the transactions contained in `RawTransactions`. This is the only part of the inner transactions that is saved as a part of the ledger within the `Atomic` transaction, since the inner transactions themselves will be their own transactions on-ledger.

### 2.5. `BatchSigners`

This field operates similarly to [multisign](https://xrpl.org/docs/concepts/accounts/multi-signing/) on the XRPL. It is only needed if multiple accounts' transactions are included in the `Atomic` transaction; otherwise, the transaction signature provides the same security guarantees.

|FieldName | Required? | JSON Type | Internal Type |
|:---------|:-----------|:---------------|:------------|
|`Account`|✔️|`string`|`STAccount`|
|`SigningPubKey`| |`string`|`STBlob`|
|`Signature`| |`string`|`STBlob`|
|`Signers`| |`array`|`STArray`|

#### 2.5.1. `Account`

This is an account that has at least one inner transaction.

#### 2.5.2. `SigningPubKey` and `Signature`

These fields are included if the account is signing with a single signature (as opposed to multi-sign). They sign the `AtomicityType` and `Transactions` fields.

#### 2.5.3. `Signers`

This field is included if the account is signing with multi-sign (as opposed to a single signature). It operates equivalently to the [`Signers` field](https://xrpl.org/docs/references/protocol/transactions/common-fields/#signers-field) used in standard transaction multi-sign. This field holds the signatures for the `AtomicityType` and `Transactions` fields.

## 3. Security

### 3.1. Signatures

The inner transactions are not signed for several reasons:
* Fewer signatures for `rippled` to process, since checking signature is expensive performance-wise
* In multi-account usage, whoever is aggregating the inner transactions from the various parties (presumably the party signing the `Atomic` transaction) cannot just submit an inner transaction without it being part of the `Atomic` transaction (since a normal transaction needs to be signed)
* Ease of use (fewer signatures needed)

The inner transactions do not need to be signed because the outer transaction is signed by all parties contributing inner transactions, so the same security principles still apply.

# Appendix

## Appendix A: FAQ

### A.1: What if I want a more complex tree of transactions that are AND/XORed together? Can I nest `Atomic` transactions?

The original version of this spec supported nesting `Atomic` transactions. However, upon further analysis, that was deemed a bit too complicated for an initial version of `Atomic`, as most of the benefit from this new feature does not require nested transactions. Based on user and community need, this could be added as part of a V2.

### A.2: What if all of the transactions fail? Will a fee still be claimed?

Yes, just as they would if they were individually submitted. 

TODO: maybe only the batch fee would be claimed? Maybe that fee should be higher?

### A.3: Why do I need to include sequence numbers in the transactions? How do I handle situations where one transaction might fail but the next might succeed?

Sequence numbers need to be included to avoid hash collisions with multiple identical transactions. Otherwise, e.g. two payment transactions for 1 XRP to the same account (for, say, a platform fee) would have the same contents and therefore the same hash.

To handle cases where sequence numbers may need to be "skipped", [tickets](https://xrpl.org/docs/concepts/accounts/tickets/) can be used.

### A.4: Would this feature enable greater frontrunning abilities?

That is definitely a concern. Ways to mitigate this are still being investigated. Some potential answers:
* Charge for more extensive path usage
* Have higher fees for batch transactions
* Submit the batch transactions at the end of the ledger

### A.5: What error is returned if all the transactions fail in an `OR`/batch transaction?

The answer to this question is still being investigated. Some potential answers:
* Return the first error encountered
* Return the last error encountered
* Return a general error, `temBATCH_FAILED`/`tecBATCH_FAILED` 
* Return a list of all the errors encountered in the metadata, for easier debugging

### A.6: Can another account sign/pay for the outer transaction if they don't have any of the inner transactions?

If there are multiple parties in the inner transactions, yes. Otherwise, no. This is because in a single party `Atomic` transaction, the inner transaction's signoff is provided by `BatchSigners`.

### A.7: How does this work in conjunction with [XLS-49d](https://github.com/XRPLF/XRPL-Standards/discussions/144)? If I give a signer list powers over the `Atomic` transaction, can it effectively run all transaction types?

The answer to this question is still being investigated. Some potential answers:
* All signer lists should have access to this transaction but only for the transaction types they have powers over
* Only the global signer list can have access to this transaction

### A.8: Why not call this transaction `Batch`?

It has greater capabilities than just batching transactions. 

### A.9: What if I want some error code types to be allowed to proceed, just as `tesSUCCESS` would, in e.g. an `ALL` case?

This was deemed unnecessary. If you have a need for this, please provide example use-cases.

### A.10: What if I want the `Batch` transaction signer to handle the fees for the inner transactions?

That is not supported in this version of the spec. This is due to the added complexity of passing info about who is paying the fee down the stack.
