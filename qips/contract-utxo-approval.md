<pre>
  QIP-XXX
  Layer: Blockchain Protocol
  Title: Reduce Block Spacing
  Author: Jordan Earls earlz@qtum.info
  Comments-Summary: 
  Comments-URI: 
  Status: Draft
  Created: 2020-11-6
  License:
</pre>

***

## Abstract ##

This QIP seeks to enable a method by which Neutron smart contract code can be used to determine if a transaction is eligible for inclusion in the blockchain. This has traditionally been possible but restricted to the limits that comes with using Bitcoin Script style "smart contracts". This QIP would enable the full power of turing-complete smart contracts written in mainstream programming languages like Rust to also be used for this determination.

## Motivation ##

Bitcoin Script style "include this transaction when something happens" smart contracts have traditionally enabled powerful, but non-intuitive methods of building scaling networks and constructing certain smart contracts which are plainly impossible to duplicate via Ethereum/Qtum style smart contracts. It is expected that including this functionality but with the powerful features that come with Neutron, that it will enable more powerful use cases for these unique construtions, and the allowance of more user friendly protocols to be built from it. 

## Specification ##

A contract "bare execution" UTXO opts in to this behavior by using a flag set in the ContractType flags, used in determining how to execute the actual smart contract code. Only one UTXO per transaction may have this behavior and the UTXO must be the first UTXO in the transaction outputs. When validating these transactions within a block, if the smart contract encounters an error or otherwise returns an ErrorCode other than 0, then the entire transaction is (at least temporarily) invalid. Thus, a block which contains such a transaction will not be valid. 

This approval UTXO is executed three times in total by a block creator, normally:

* Once when accepting the transaction into the mempool. If the Approval UTXO gives an invalid result, then it will be rejected for mempool inclusion
* Once when creating a new block. If the Approval UTXO gives an invalid result, then the transaction will not be included in the block
* Once when accepting the new block into the node's blockchain. If the Approval UTXO gives an invalid result (should be impossible by design, but still checked) then the block is invalid and rejected.

For non-relaying nodes and validators, thie feature shall incur no additional resources nor economic consequences. Validating the UTXO Approval should not be anymore expensive than validating any other smart contract transaction. 

For relaying nodes and block creators however, the behavior to accept such transactions shall be opt-in. These nodes that do not opt-in will not incur any additional resources of economic consequences. For nodes which do not opt-in, they will reject all such transactions from being included in their mempool and not attempt to create new blocks containing such transactions. However, they must still validate the transactions that are put into blocks by other nodes.

The opt-in process is a soft network feature which can be modified extensively as needed without the need to introduce consensus behavior changes or forks. Some potential options for controlling the extent of opt-in:

* Maximum gas limit for an Approval UTXO
* Maximum number of Approval UTXO transactions within the mempool or mined into a single block at a time.
* Only accept certain hashes of Approval UTXO code, or more complex parsing of these transactions to only accept them if they contain a particular signature etc
* Work from a whitelist of approved senders for Approval UTXO transactions
* Only accept certain VM types or execution flags

The on-chain protocol only has the following restrictions:

* The Approval UTXO must be a bare smart contract execution
* The Approval UTXO's gas limit must be less than the block gas limit

## Compatibility ##


## Nongoals ##

* Although this provides speculative execution of smart contract code on the blockchain, this is not designed to allow for "free" or feeless transactions from the point of view of the blockchain. Someone must still pay a fee unless the block creator is specifically accepting 0 fee/0 gas price transactions

## Acknowledgments ##


## References ##



## Copyright ##


