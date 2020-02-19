
- Title: improved-wallet-state-management
- Authors: [Michael Cordner](mailto:yeastplume@protonmail.com)
- Start date: Nov 4th, 2019
- RFC PR: [mimblewimble/grin-rfcs#30](https://github.com/mimblewimble/grin-rfcs/pull/30)
- Tracking issue: [mimblewimble/grin-wallet#244](https://github.com/mimblewimble/grin-wallet/issues/244)

---

# Summary
[summary]: #summary

The changes outlined in the RFC are intended to make process of updating a wallet's state from the chain more consistent for developers and more transparent to the end user. 

This includes the following changes:

* The wallet's update process is modified to be more consistent and encapsulated.
* The `check-repair` process is run periodically as part of normal wallet update operations
* The ability for the wallet to run the update process on a separate thread is added.
* A TTL (Time-to-Live) field is added to the slate
* The default output selection method is set to `smallest`

# Motivation
[motivation]: #motivation

Grin wallet previously updated the state of its outputs and transactions using a combination of UTXO updates, Kernel Lookups and UTXO set scanning. These 'primitives' on their own are generally enough to keep wallet states consistent, however the manner in which they were previously invoked was less than ideal and relied heavily on manually invoking the `check-repair` process and `cancel` command.

To rectify this situation, this RFC outlines enhancements made to the wallet update process that have the goal of ensuring wallets are always in a consistent state in a manner that is transparent to the user. After adopting these changes, most wallets should automatically keep themselves in sync with the chain in all cases, and users will not usually have to invoke the `check-repair` or `cancel` commands.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

While these changes should be transparent to end users, wallet developers should note the changes to how the wallet updates itself via these processes. Special attention should be paid to the new update thread API functions, as well as the new TTL field and default selection strategy change.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Wallet Update Procedures

The following sections outlines the processes used in the creation and update of outputs and related transactions within the wallet.

### Transaction and Output Creation

#### Coinbase Output + Transaction Creation

Coinbase outputs and their related transactions are created via the wallet listener as part of the mining process. The workflow is:

 * When mining, create a potential coinbase output for the target block.
 * If the block is accepted and the output is detected in the UTXO set (via the [Update by Output process](#update-by-output-process), create a transaction log entry of type 'Confirmed Coinbase', with 'confirmed' set to true.

#### Transacting (Payer -> Payee) Output + Transaction Creation

Outputs and tranasctions are created during the transaction exchange process as follows. Note that the invoicing workflow is mostly identical with the roles of Payer and Payee reversed, so this workflow is not outlined separately.

 * The sender creates a new 'blank' transaction Slate, adding inputs and change outputs to the slate.
 * The sender generates a random kernel offset. (The kernel offset is always generated by the slate creator)
 * The sender sends the slate to the payee (via file or http). If sending synchronously, (e.g. via http)  the associated transaction is saved to the log after a response from the payee's listener. If sending asynchronously, (e.g. via file), the transaction is saved immediately. 
 * When saved, the associated transaction is set to type 'Sent Tx' with a status of 'unconfirmed', inputs are locked internally and a change output is added with status 'unconfirmed'.
 * The payee receives the slate, creates (an) output(s) for the received amount with status unconfirmed, and immediately stores a transaction in their log of type 'Received Tx' with confirmed set to false. 
 * The payee calculates and saves the transaction kernel commitment for later reference.
 * The slate is returned to the payer for completion, who calculates and saves the transaction kernel. The transaction is sent to a node for validation.

### Transaction and Output Statue Update Processes

#### Update by Output Process

The Update by Output Process is the main method by which the wallet updates its internal state, decides whether a transaction has been confirmed and when to remove or confirm individual outputs. This process works as follows:

* For every unspent output in the wallet:
* If the output's status in the user's wallet is 'unconfirmed' and it is present in the UTXO set, change the output's status to 'confirmed', and update the associated transaction status to 'confirmed'.
* If the output's status in the user's wallet is 'confirmed' and it no longer appears in the UTXO set, set its status to 'Spent'. Note these outputs will usually be locked so they cannot be selected for spending again.

#### Kernel Lookup Process

It is also possible to look up a transaction via the kernel that was stored when the transaction and outputs were created. This is necessary in cases such as where a participant in a transaction doesn't have any change outputs, which means the output update process won't detect when to mark a transaction confirmed. This process is:

* Retrieve the transaction kernel from the node using the kernel excess value calculated during the transaction creation process.
* If the kernel exists, update the status of the associated transaction to 'Confirmed'

#### Check-Repair Process

The Update by Output process on its own is not enough to ensure the contents of a wallet is correct. There are many easily-encountered situations in the course of a wallet's operation where this process is insufficient and can potentially lead to an wallet state that's inconsistent with the UTXO set, including but not limited to:

* Manually cancelling transactions and unlocking outputs before they've had a chance to confirm on the chain.
* Running multiple wallets from the same seed
* Fork situations

The check-repair process fixes most of these potential issues by scanning every single output in the UTXO set and testing for ownership by determining if its bullet proof can be decoded using the wallet's master key. It will then 'check and repair' all outputs and transactions in the wallet as follows:

* If an output exists in the UTXO set and is not in the wallet, create an output for the wallet at the derivation path stored in the bullet proof, and create a new transaction crediting the amount
* If an output exists in the UTXO set and is marked as 'spent', mark it as unspent and cancel any associated transaction log entries.

Additionally, the check-repair process can take a flag instructing it to delete unconfirmed outputs and reset all outstanding transactions. If this flag is set:

* If a locked output exists in the UTXO set, unlock it and set any associated transactions to `cancelled`
* If an 'unconfirmed' output is not in the UTXO set, delete it and cancel any associated transaction log entries.

Note that the wallet `restore` process works very similarly to the check-repair process, however it must always operate on an empty wallet meaning it is limited to creating outputs and transactions in the wallet.

## Previous Overall Update Process

All of the procedures outlined above can be considered set of 'primitives' available for the wallet to keep itself updated. The previous update process was somewhat limited, and had the tendency to require a significant amount of manual updating using the `check-repair` command. The previous process was as follows:

* Invoke the 'Update by Output' process before retrieving any info relating to the wallet state (`txs`, `info` or `outputs` commands,) or creating a new transaction. This invocation is usually done internally and in a syncronous blocking manner by each command that requires the wallet state to be as up to date as possible.
* During the `txs` retrieval command, for each unconfirmed transaction with an `amount recieved` field set to 0, invoke the 'Kernel Lookup' process.
* If the user suspects the wallet state to be inconsistent with the node's UTXO state, the user can run the manual `check-repair` command. This will scan all UTXOs in the set from position 1 in the output PMMR. The user can optionally provide a flag to remove all unconfirmed transactions.
* If a transaction and associated outputs appear 'stuck' or 'locked' due to the other party not completing a transaction, the user must manually cancel the transactions and unlock the outputs for re-use. (or provide the `cancel-unconfirmed` flag to the check-repair process)
* The default selection strategy is to 'sweep' the wallet of outputs on each transaction creation, meaning that by default, no further transaction can usually be made while a transaction is outstanding.

## Changes to Overall Update Process and Procedures

While a combination of the 'primitives' listed above should be enough to keep a wallet's state consistent with a node's UTXO set, the previous method of invoking the vital `check-repair` command was a manual step that had to be run on the entire UTXO set each time. The overall update process is modified to incorporate the previous `check-repair` logic as part of normal operation. To ensure a wallet is only scanning the part of the UTXO set required, the wallet stores details on what parts of the UTXO set it has already scanned, and performs incremental scans as part of normal update operation.

### `check-repair` is renamed to `scan`

The naming of the previous `check-repair` command and functionality implied something had gone wrong, whereas it really should be considered a necessary part of normal wallet operation. It is therefore renamed to a more friendly-sounding `scan` process.

### Overall Update Process

The Overall Update process is outlined as follows:

* Perform the 'Update by Output' process.
* Perform the 'Kernel Update' process for transactions where the incoming amount is 0.
* Query the wallet's internal data for the last block height scanned via the `scan` process. 
   * If the stored block's header hash doesn't match the version currently on the node's UTXO set, scan the range of the UTXO set corresponding to the last stored block - 100 blocks up until the current block.
   * If the stored block's header matches the version on the chain, scan a range of the UTXO set corresponding to the stored block height to the current block height.
* Save the last block scanned by the `scan` process.

### New / Recovered Wallets

Newly created and recovered wallets scan the UTXO set as follows:

* New wallets generated with a new random seed will mark themselves as `new`, and set their `last scanned block` to the block height reported on first successful contact with a node.
* Wallets recovered with an existing mnemonic will set the `last scanned block` to `1`, triggering the `scan` process and thereby restoring any associated outputs.
* The `restore` command will be removed in favor of the `scan` process.

### Manual Scans

It is still be possible to manually scan the chain via the `scan` (`check`) command, however the command is modified to allow the user to specify a range of block heights to scan. If this is not provided, the scan occurs from the current block - 1 week's worth of blocks.

### Invocation of Update Process

The update process was previously run inline in a synchronous blocking fashion as part of individual wallet commands. With the `scan` command becoming part of normal operations, it's expected that a particular invocation of the overall update process could potentially take a long time. This is acceptable in single-use modes such as the single invocation model of `grin-wallet`, but is far less usable in environments where the wallet and Owner API stay resident and the caller may not have any particular insight as to why a call invoking the update process might be taking a long time to complete.

To address this, the wallet provides a method of calling the Overall Update Process in its own thread that attempts to minimise its usage of the wallet lock. The Wallet's Owner API (V3 Only) is extended with the following functions:

* `start_update_thread(interval)` => Starts the update thread, calling the Overall Update Process at the frequency specified
* `stop_update_thread()` => Stops the update thread
* `get_update_status()` => Returns the current status of the update thread, along with as much information as possible about what the thread is doing and, in the case of the `scan` process, the percentage complete.

Further, if the update thread is currently running, commands that previously always called the Overall Update Process (such as `info`, `txs`, etc...) do not call the update process. This means that when invoked via the command line, the behaviour is unchanged. However, wallets running the Owner API can instead choose to run and monitor the background update thread at the desired frequency while keeping the user informed of any long-running update processes.

### Transaction TTL

The Update process outlined above does not cover the case where a slate is exchanged and outputs are locked, but one party doesn't complete the transaction for whatever reason. 

Previously, a situation such as this meant that the outputs associated with a transaction sat locked indefinitely within a user's wallet, with the only way to unlock them a manual `cancel` command (or a `check-repair` with the `delete-unconfirmed` flag set). Worse, the wallet's default selection strategy was set to `all` meaning an entire wallet's contents usually became locked on each transaction, and thus a single uncompleted transaction would render a user's entire wallet balance unspendable.

To rectify these problems, the following changes are made:

* A Time-To-Live (`TTL`) field is added to the Slate, which is defined as the last block height at which a wallet should attempt to complete a transaction. This field should be respected at the wallet-exchange level, but is not currently commit to at a consensus level. If a wallet detects a particular transaction's TTL has expired, it will automatically cancel the transaction and unlock associated outputs. Note this doesn't prevent the other party sending the transaction to the node after this happens, but there is no guarantee the outputs will still be available. However, if this does happen the `scan` command will correct the wallet state.
* The default output selection method is changed to `smallest`, to prevent all wallet amounts being locked on every transaction. The `smallest` strategy prefers using the smallest (or 'dust') outputs as inputs to a transaction, and it is conjectured that the overall effect on Grin's UTXO size will be negligible in the longer term, provided users continue to transact. The `all` method of output selection also has privacy implications in that it makes it easier for an observer to identify a group of outputs as belonging to a single wallet (or indeed, representing the entire contents of a wallet in most cases).

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Do we limit the Kernel Lookup part of the process to just cases where there is no change output. Is there any benefit to calling it for all outstanding transactions on each lookup (or any particular downside to doing so?)
- What is the effect on UTXO set size of changing the selection method to `smallest`?
- Confirm sensible defaults for how far to scan back on a `scan`