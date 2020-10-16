# NeutronDB Overview

NeutronDB (previously "DeltaDB") works on the concept of trees of change logs or "deltas". Specifically, by reading up the entire series of deltas across the entire blockchain, the entire state of the system can be built up. However, because the deltas also track when data is read, it is also possible to simply "forget" old trees of deltas and to still have a partial state which can continue to be used and modified. With this, a rent system can be built on top of this architecture extremely easily, and exposed to smart contracts in a "black box" manner. Specifically, smart contracts should not be at all aware of the rent status of their various pieces of state within the database. Instead, if an item which is out of rent is attempted to be read, then the smart contract execution immediately terminates as a "non-recoverable" error, meaning that the entire chain of smart contract execution is terminated and no state persisted. A simple mechanism is provided for restoring state which has fallen out of rent, whereby the state and certain proof information is attached to the smart contract transaction.

The overall goals of such a database system are the following:

* Ability to store an arbritrary amount of data without complex packing or unpacking logic needed, ie, no 256 bit word limit
* Only the most often used data should be kept within the rented database. This ignores financial incentives of keeping certain smart contracts alive and instead prioritizes to only keep what is actually needed on nodes based on actual blockchain usage
* Incurs near zero cost on the actual writing of smart contract code. Smart contract code is already quite difficult to write with many different security concerns. Adding rent concepts that the programmer must manage and account for is just making this even more difficult. 
* Bytecode and smart contract state should be treated the same as far as underlying data structures and consensus proofs. This means that there is less special case logic needed, and bytecode could be loaded and modified by smart contracts in creative ways. 
* Censorship resistance and detection built into the proof system that can be utilized by SPV and light wallets

The overall structure consists of a DeltaRoot merkle tree with "staggered" data:

* Smart contract address A
* AddressDeltaRoot for address A
* Smart contract address B
* AddressDeltaRoot for address B
* etc...

The AddressDeltaTree is then another merkle tree specifically covering only a single address's state and consists of:

* Delta 1 (change log/read report)
* Delta 2
* Delta 3
* etc...

A Delta can encompass a number of different types of information. This includes:

* State creation/modification
* State read
* Execution receipt
* "Log" data (similar to Ethereum Events)

Note that smart contracts normally can get no real insight into the current status of the Delta tree system, however, allowances are made for reading execution receipts and log data from smart contracts. Note that the ability to read such info is limited to only be within the rent period. Unlike normal state, these special pieces of state can not be restored once they fall out of rent, and furthermore will not be propagated by being read. The only insight that might be given to smart contracts is the ability to get previous block header information, which would include the DeltaRoot hash which can be used for verifying proofs. 



# The old state restore problem

One issue exists with the state database currently, one that is an implementation detail and not necessarily a blocker for the consensus proofs. 

Specifically, the issue is that there must be a mechanism to detect if an old state is attempted to be restored when a new state already exists, and in that case to reject the restoration as invalid.

* At block 100, state A set to 10
* At block 200, state A set to 20
* Many blocks pass and state A falls out of rent
* Restoration is attempted using proofs from block 100, to restore A to 10
* Because nodes lack information about block 200's trees after rent has passed, how can this restoration be stopped?

## The naive solution

The simplest solution to this issue is to track the block number last set for each state key:

* At block 100, state A set to 10, A's last block height set to 100
* At block 200, state A set to 20, A's last block height set to 200
* Many blocks pass and state A falls out of rent, however, the piece of data for A's last block height is still tracked and thus set to 200
* Restoration is attempted using proofs from block 100, to restore A to 10
* Node tracks that A's last block height was 200 and thus the proof from block 100 is invalid

This is a simple solution, but introduces several inefficiencies:

* Each state key must still be tracked, meaning there is still an unbounded growing size, especially given that the state keys can potentially be large
* This is very wasteful in the case of small pieces of state. For instance, if you have a key named "enabled" with a boolean state, then it actually costs more disk space to track the last block height than the actual data. Given previous ways that Ethereum contract state tends to be structured, small pieces of state is expected to be a fairly common case. 

## Reject Set solution

In this method, some minor consensus proof modification would be necessary. The basic workflow:

* At block 100, state A set to 10, "tracking ID" for A set to 1
* At block 200, state A set to 20, "tracking ID" for A is set to 2, and tracking ID 1 is added to the reject set
* Many blocks pass and state A falls out of rent
* Restoration is attempted using proofs from block 100, to restore A to 10. 
* Because the block 100 proof for A has a tracking ID of 1, it is detected as invalid by looking it up against the reject set

This would seem similar to the naive solution, however, consider it with many more pieces of state:

* At block 100, state A set to 10, "tracking ID" for the proof is set to 1
* At block 150, state B set to 50, tracking Id for the proof is set to 2
* At block 200, state A set to 20, "tracking ID" for the proof is set to 3, and tracking ID 1 is added to the reject set
* At block 300, state C is set to 80, tracking ID for the proof is set to 4
* At block 350, state B is set to 100, tracking Id for the proof is set to 5 and tracking ID 2 is added to the reject set
* Many blocks pass and state A falls out of rent
* Restoration is attempted using proofs from block 100, to restore A to 10. 
* Because the block 100 proof for A has a tracking ID of 1, it is detected as invalid by looking it up against the reject set

Thus, in this method there is no need for tracking every key indefinitely like in the naive solution. Instead, each state modification delta will have a tracking ID that is a simple autoincrementing number. When a state is modified, the previous tracking ID must be known (ie, the state must be within rent) and that tracking ID is then thrown into the reject set. The reject set is then a simple list of integers per address. The total size of the reject set is still unbounded, but can only grow by a small amount at a time. This could potentially even by futhur reduced by using a series of perfect bloom filters or some other probabalistic data structure, but this requires a lot more thought and is really more of an implementation detail as it would not affect consensus proofs. 

There may be additional security for SPV and light wallets as well by somehow putting a hash proof of the reject set into the block header. Using a list of block headers, it would then be trivial to detect stale and false state proofs without needing to fully verify the blockchain.

This approach does come with one downside however. In the case of state modification, the tracking ID of the old state must be known. This means that it is not possible to simply replace out of rent state and instead that state must be restored prior to replacement. There could potentially be a short cut to this, such as submitting a proof for the tracking ID without needing anything more than a hash of the actual state, however, it is still a slight inefficiency. 





