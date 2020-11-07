Note: This is copy-pasted from an old document and needs updated into a proper QIP format and also the AALv2 proposal will be split to a separate proposal




# Segregated design

Neutron operates completely independently of the legacy EVM infrastructure, with the exclusion of OP_SENDER being supported (probably). 

The following new Script opcodes are introduced:

* OP_NDEPLOY -- Deploys a new Neutron contract
* OP_NCALL -- Calls an existing Neutron contract
* OP_NEXEC -- Executes arbritrary Neutron code which is not saved to state (can be used for complex and dynamic calls)
* OP_NSPEND -- used by AALv2 for spending UTXOs (?)
* OP_NGASFEE -- used by AALv2 for tracking gas fees
* OP_NBALANCE -- used by AALv2 for internal tracking of contract balances.

As much work as is reasonable will be off-loaded into Rust Neutron code. Obvious things to implement in C++ though include anything involving transaction writing (ie, writing the Bitcoin Script etc), block database access, UTXO database access, etc. Ideally this method should more cleanly separate Neutron from Qtum Core, making the implementation of both more clean and logical. 

## Neutron Fail-Safe

There shall be a "fail-safe" method of excluding most Neutron code (specifically everything from Neutron returning "null") but otherwise unaffecting Qtum's consensus. This may be implemented as a client-side hidden command line argument in Qtum Core. The following hidden arguments are proposed:

--disable-neutron-staking -- this will disable all Neutron processing in block creation and will not accept nor relay Neutron transactions in the mempool network. This is for use where a bug is found which causes block creation with Neutron transactions to fail, to prevent block creators from failing to move the network forward. This may trigger some kind of additional network message which can be used to warn of potential network problems to non-block creating clients. 

--disable-neutron-validation=[block_number] -- this will disable all Neutron consensus processing (resulting in block rejection) after the specified block number. Of note is that previously verified blocks will not be removed from the client's view of the blockchain unless block_number is specified to be before the block. This is to be used only in case of dire emergency where a Neutron bug has been found to cause a failure of block validation consensus. It would be possible to distribute a block_number value to use to keep the network moving and consensus working while a fixed client is worked on. This must come with a huge warning that using this value without coordination will cause the client to fork from the network. Of note is that Neutron block validation errors using this option should not trigger a DoS error style ban. 

If either argument is used, all read-only Neutron RPC calls will continue to work as normal, but transaction creating Neutron RPC calls will return an error. 


## AALv2

The AALv2 will fix a number of issues that were identified in the old AALv1. Specifically the biggest changes include:

* The actual implementation of the AALv2 will work only from a structure containing the UTXO addresses to be spent, UTXO addresses/balances to create, and the amount of gas to refund and to what address. It will have minimal knowledge of the internal workings of Neutron. Balance mapping will be implemented within Rust Neutron code, while creating condensing transactions will be implemented in C++ Qtum Core code
* When a contract is deployed, the UTXO used will immediately be spent into an AAL condensing transaction. If there is no balance for the contract, it will effectively have no UTXO (is this a problem?). If there is a balance, then it will be an output using OP_NBALANCE
* If a contract receives Qtum using OP_NCALL/OP_NEXEC which previously had none, and no balance changes occur in the block, this UTXO will still be spent and re-output as OP_NBALANCE. This is to keep the UTXO set as small as possible and save space. Call data is not important to preserve
* A contract can be sent Qtum without actually executing the contract using OP_NBALANCE. This will trigger a condensing transaction if the contract already has a non-zero balance. 
* Gas fees will use an explicit UTXO rather than the old method of overloading it into the Bitcoin style "left over fee" system. This means users can immediately spend gas refunds and that we can be more flexible with how gas fees are handled, such as allowing for a different address than the sender to receive gas refunds, or for gas refunds to be split across multiple addresses
* Sorting order of UTXOs spent and created are explicitly sorted so that implementation changes and optimizations are greatly simplified while maintaining consensus
* The AALv2 will be designed in preparation of UTXO-states being implemented (TODO)
* The AALv2 will build a single large condensing transaction per block, rather than per contract transaction. The size of the transaction can be easily calculated after each contract transaction to be certain that the condensing transaction will not cause the block to exceed the maximum block size. 
* The AALv2 condensing transaction must be the last transaction of the block. 

### Neutron Communication

The AALv2 will accept a structure of the follow type from Neutron to build the condensing transaction:

-- "utxo_pair" is a type defined as pair(txid as [u8; 32], vout_number as u32)

* addresses_modified[UniversalAddress as [u8; 24]] -> pair(pair(utxo_pair, old_balance as u64), new_balance as u64)
* gas_refunds[UniversalAddress as [u8; 24]] -> amount as u64
* state_utxos_spent[u32] -> utxo_pair
* state_utxos_created[u32] -> ???

The AALv2 will be expected to provide the following functions to Neutron:

* committed_balance_of_contract(address: UniversalAddress) -> pair(u64, utxo_pair) -- if balance is 0, the utxo_pair will be null.
* utxo_unspent(utxo: utxo_pair) -> pair(unspent as bool, amount as u64) -- not strictly used for accounting purposes, probably optional
* state_utxos_of_contract(address: UniversalAddress) -> ??

Neutron will be expected to provide the following function:

* estimate_condensing_transaction_size() -> u64 -- This function may not be perfectly accurate but MUST not return a size smaller than the actual size

This function is needed because Neutron does not return the final condensing transaction info structure to the AALv2 until there is no more transactions to add to the block. Creating the info structure may be somewhat expensive (due to conversion from Rust structures to C compatible structures) and so should not be done after every contract transaction. 

### Condensing Transaction Size Estimation

Neutron will be aware of how much data each type of address requires to emit an output for. It will thus count all of the addresses_modified pair accumulating for each UniversalAddress's actual transaction size. In addition it is known that each UTXO spent requires about 40 bytes of data since all UTXOs spent should only consist of an OP_NSPEND opcode and no other data. Each gas refund is also counted as is typical for an additional output. 

### Sorting

All UTXOs spent and created within a condensing will be done in an explicit order. 

The order is first sorted by the following address types:

* Neutron Balances
* Neutron state UTXOs
* OP_NGASFEE (input only)
* pubkeyhash
* scripthash
* segwit-scripthash
* non-standard transactions (if supported)

Within each type of address, the address is then sorted by the hex code of each address (or for non-standard transactions, by hash of the resulting script code)

### Gas Fees

In AALv2 gas fees will use an explicit UTXO. Specifically, this could look like so:

Existing State: contract X holds 5 Qtum as its balance.

Inputs:
* PKH A - 10 qtum
* PKH B - 5 qtum

Outputs:
* PKH C -- 2 qtum ("change" utxo)
* Neutron Call to X -- 0.5 qtum
* OP_NGASFEE output of 12 qtum

Remainder fee: 0.5 qtum, considered a "tip" fee to block creator. Completely optional

For this example, the call had a gas price and usage which resulted in 3 qtum being used by gas costs, thus there is a 9 qtum gas refund. Neutron controls what addresses get a refund, potentially including potentially splitting of fees between multiple addresses. This may be useful in the future, but no hard plans are made yet. For this simple example though, all gas refunds are returned to PKH A

The resulting condensing transaction would look like so:

Inputs:
* Contract X old UTXO representing it's balance -- 5 qtum
* Neutron Call to X from previous transaction -- 0.5 qtum
* OP_NGASFEE output from previous transaction -- 12 qtum
(total spent: 17.5 qtum)

Outputs:
* Contract X new UTXO representing balance -- 5.5 qtum
* PKH A -- 9 qtum
(total output: 14.5 qtum)

Remainder fee: 3 qtum, rewarded to the block creator like a typical bitcoin style fee

Note: if a particular address is listed as receiving money due to a balance modification, and also should receive a gas refund, these two amounts will be combined into a single UTXO. Excluding potential state UTXO stuff, no address should ever be sent more than one UTXO per condensing transaction. 

### Contract Error Refunds

In the case of a contract transaction which sends Qtum to a contract but the contract execution involving it reverts, the AALv2 responds similarly as v1. Specifically the output will still be spent, but will returned as a new output created in the condensing transaction. 

Note: this new duplicated transaction output can not include some things such as the same locktime.

### Transaction Templates

Contract Call:

```
contract address
neutron data 1
neutron data 2
neutron data n...
OP_NCALL
```

Contract Deployment:

```
neutron data 1
neutron data n...
OP_NDEPLOY
```

Bare contract execution:

```
neutron data 1
neutron data n...
OP_NEXEC
```

Balance assignment by condensing transaction OR by bare send without execution:

```
contract address
OP_NBALANCE
```

Spending scriptsig for condenscing transaction spends:

```
OP_NSPEND
```

Gas fee utxo:

```
neutron gas transaction version (treated as neutron data 1)
gas price (treated as neutron data 2)
(optional) neutron data 3 (TBD)
OP_NGASFEE
```

### Fee Parsing

Neutron is called for interpreting the gas fee. The first 3 items of neutron data in the Gas Fee UTXO shall be pushed to Neutron's Smart Contract Call Stack (SCCS). The number of items pushed may be changed using a non-forking standard transaction addition based on the Neutron Transaction Version. For example, it is possible to add a version of 3 which indicates that 5 items should be pushed for use by Neutron without a fork 'in theory'... however, any significant Neutron update like this would typically involve a fork. 

The expected data to be contained and interpreted by Neutron:

* Neutron Transaction Version (1 byte), currently shall be 1. Expected to also include "2" for potentially "free" transaction QIP, to be published later
* Gas price (8 bytes) -- This is priced at 0.01 Qtum satoshis for better granularity. Any non-integer in the final Qtum price is rounded up to the nearest satoshi

The exact function to be used shall be of the format:

neutron_parse_gas(self, sccs, blockchain_info) -> (valid, gas price, gas limit)

In the future something like escalator gas, or EIP-1559 may be implemented and this is designed to be flexible enough to allow this. 

### AALv2 Special Rules

Neutron execution containing transactons must have the following rules:

* At least 1 input
* At least 2 outputs of which at least 1 output is a neutron execution (op_ncall, op_ndeploy, op_nexec, op_nbalance)
* Exactly one gas fee output. This output determines the gas limit and gas price for the complete transaction.
* op_nbalance will incur a fixed gas cost subtracted from the total gas stipend
* No neutron execution can specify a gas limit greater than that paid for by the gas fee output
* Each neutron execution may together have a gas limit which adds up to be a higher gas limit than the total gas fee. The latter executions however may receive a lower effective gas limit than requested, potentially resulting in failure of the execution

The condensing transaction must have the following rules applied:

* Each input must be spent using exactly OP_NSPEND and nothing else
* Each input must be in the sorted order as described above
* There must be at least 2 outputs and at least 2 inputs, if there is a condensing transaction at all. 
* Each output must be in the sorted order as described above
* Each output address must be mentioned exactly once in a condensing transaction
* All OP_NGASFEE outputs in the block must be spent by the condensing transaction
* All neutron execution outputs in the block must be spent by the condensing transaction
* 

## Block Proofs

The "utxoRoot" field in the Qtum block header which is currently used for the EVM will be replaced by the Neutron field deltaTreeRoot. The utxoRoot has been determined to be unnecessary for security and very rarely, if ever, used for client integrations. Thus, it can be repurposed to avoid a format change to the block header, which would require a fairly significant increase in implementation difficulty. 

There are no other block proofs determined to be necessary. The AALv2 will work perfectly fine without any tree or state structure tracking every contract UTXO. Instead, the simple UTXO set searching functionality already built in will be utilized for locating smart contract balance UTXOs. If the UTXO set searching is not efficient enough for this, we can look into adding a new UTXO index similar to address-index but only for smart contract addresses. This could also be done after forking etc in response to problems that only arise in major usage.



## Neutron Context

An amount of data is given to initialize Neutron for contract execution. This data is fed specifically into "Qtum-Neutron-Bridge", a project which implements the Neutron System Call layer and translates the blockchain data into a form Neutron contracts can interact with. There are also some calls that will require Neutron directly calling back into Qtum:

* get_historical_block_hash(height: u32) -> u256: hash
* get_historical_block_info(height: u32) -> BlockInfo
* committed_balance_of_contract(address: UniversalAddress) -> pair(u64, utxo_pair) -- if balance is 0, the utxo_pair will be null.
* utxo_unspent(utxo: utxo_pair) -> pair(unspent as bool, amount as u64) -- not strictly used for accounting purposes, probably optional
* state_utxos_of_contract(address: UniversalAddress) -> ??

The context will otherwise be split into 3 sections:

* Block information
* Transaction information
* Execution information

Note this context info will be exposed to Neutron Contracts primarily through system calls. All Neutron decided information is thus not included here. This only covers what Qtum is to give to Neutron

### Block information:

* Block hash of previous 16 blocks (more available through get_historical_x)
* Difficulty of current block
* Time of current block
* Current block gas limit (DGP?)
* Current block height
* Block creator address (if created by a super staker, this would be the UTXO address responsible for the coin stake)
* Super Staker delegate address (if created by a super staker, this would be the pool address)
* Super Staker fee percent (if created by a super staker, the % fee declared that must be paid to the super staker out of the block reward)

### Transaction Information:

* Array of inputs
* Array of outputs
* Array of Neutron Transactions within the outputs
* Block creator "tip" fee
* Lock time
* Effective gas price

Input info:
* Address
* Value

Output info:
* Address
* Value

Neutron Info:
* Output number
* Type (either deploy, call, bare, gasfee, or addbalance)
* Array of `Vec<Vec<u8>>` where each item is the bitcoin script items leading up to the OP_Nx opcode

### Execution Information:

none? All Neutron decided?


## RPC Calls

TBD

