# Thoughts on Neutron use cases

Here are a few thoughts where I basically try to figure out in actual detail how Neutron's various features could potentially be used for either improvements in existing blockchain projects, or used to create currently-impossible projects.

These are just thoughts and not whitepapers, so there clearly could be some holes in them. This is more to solidify why each Neutron feature should exist, and why it's worth putting in the effort to include.

Unless noted, these are assumed to be with Qtum's integration of Neutron


# Off-chain ERC/QRC20 Microtransactions

This would enable to QRC20 tokens to be used for unidirectional payment channels. This is based on the Jeremy Spilman style payment channel concept for Bitcoin Script.

First, a "backup refund" contract approval transaction is constructed but not broadcast:

* Transaction is not valid until EXPIRE_TIME
* (optional) OR Transaction is valid when QRC20 2 signature construct returns that any amount has been redeemed
* (optional) AND Attached nonce matches the nonce on the 2 signature construct
* Approval contract contains a nonce
* Another output is created which will return all remaining payor QRC20 funds and removing the 2 signature construct. This requires signatures from payee and payor before continuing

Second, a locking transaction is constructed and broadcast:

* Payor includes an output which locks a set of their QRC20 tokens into a 2 signature (payor and payee) required construct, to prevent them from being spent outside of the payment channel

Third, a mutable payment transaction is constructed by payor and sent to payee but not broadcast:

* Payor includes an output which includes a signature to authorize their part of the 2 signature construct, signing the amount to be paid, in terms of "total amount that can be withdrawn over the lifetime of this construct"
* Payee can then sign this transaction, broadcast, and get their funds. They can do this with future transactions as well without destroying the payment channel
* (optional) payment includes a nonce incrementing operation, which would invalidate the previous backup refund and thus require a new backup refund transaction to ensure the payor is safe.

Operation:

* Both parties hold backup refund, it is not broadcast unless payee is unresponsive
* Locking transaction is broadcast after the backup refund is established
* Payment transaction is reconstructed by payor and sent to payee after each block of work/service
* If payee and payor agrees to allow for an early payout, then payor requests a new signed backup refund using the next nonce value from the payee. They may also negotiate a new EXPIRE_TIME. The next payment transaction will pay out the owed balance as well as increment this nonce. Note that this will not finalize nor close the payment channel
* If payee becomes unresponsive, backup refund can be broadcast at anytime after EXPIRE_TIME
* If payee decides to cash out without negotiation with payor, then payor can exit the agreement at anytime after this takes place.

Note that all of these transactions in UTXO form could use no actual inputs other than an OP_SENDER "from nowhere" UTXO. In which case, no gas price would be specified within the transaction, and no careful UTXO locking is needed. Whoever broadcasts the transaction would be expected to attach an input to be spent for fees and attach a gas price specifying UTXO


# Fast centralized side-chain with decentralized security assurances

This would enable for a fast side-chain which includes limited smart contract functionality on the side-chain. In this example, the side-chain is operated solely by a single centralized operator which signs every transaction on the side-chain. There are no formal blocks and it is rather a chain of single transactions building up the world state of the side-chain.

The limitations:

* Contracts can not call other external smart contracts
* There is no built-in currency for the side-chain, thus everything is a smart contract execution
* Each smart contract execution must abide by the "effective" block gas limits on the parent chain
* Each smart contract deployed to the side-chain must first be deployed to the parent chain
* Some interaction is required due to input state being dependent on other executions. In otherwords, transactions can easily become invalid and will need to be rebroadcast for busy contracts

Note these limitations could be solved with further thought and complication, but is not relevant to the core idea.

Note that smart contracts here are typical Neutron smart contracts without special API usage etc other than to abide by these restrictions.

The side chain transaction could look like so:

* gas limit of execution (when broadcast)
* execution address (when broadcast)
* ABI data (when broadcast)
* Signature of user (when broadcast)
* Signature of operator (when accepted)
* Utilized input state (as a merkle tree etc) (when broadcast)
* Difference in output state (when accepted)
* nonce of side-chain interaction for the execution address (when broadcast)

State can be locked on the parent chain and "redeemed" on the side-chain. The fate of that locked state is determined by the side-chain. 

For example, take a simple QRC20 token transfer:

* user would submit on-chain transaction to transfer QRC20 to side-chain state contract
* side-chain operator would then issue a side-chain transaction sending that QRC20 to the user's address

With no conflicts, there is no need for parent chain transactions other than to lock and unlock state.

When a conflict arises, it can be handled using parent chain transactions that effectively audit the centralized operator.

## Operator falsifies a side-chain transaction result

In this example, the operator modifies the output state merkle tree to give an incorrect value of some sort. This could include invalid state data, false execution result, etc.

The user sees this and can now move the action to the parent chain in order to validate the actual result. The user does this by constructing a parent chain transaction that does the following:

* Calls the operator validation smart contract with the transaction data, and the utilized input state data
* The operator validation contract ensures the signature of the operator is valid and that the utilized input state matches the tree in the side-chain transaction
* The operator validation contract executes a "virtual call" using the side-chain data
* The virtual call will execute it in an identical way that the side-chain would. It then gives the difference in output state as a result
* The operator validation contract compares the executed output state to the side-chain transaction. If this fails, then it proves the operator was wrong. This could trigger punishment mechanisms etc

## Operator falsifies parent chain state being loaded to side chain

???

## Operator falsifies side chain state being loaded to parent chain

This is basically the same as the falsification of a side-chain transaction result. It would involve a commit-challenge period on the state redemption contract on the parent chain



# Atomic swaps and tokens and state for Neutron integrating blockchains









