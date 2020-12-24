
# Component Detail

The following components will be included within this spec:

* CoMap -- a set of communication maps for smart contracts talking primarily to external smart contracts and vice versa, an integral piece of NeutronABI specifications
* CoStack -- a set of communication stacks for smart contracts talking to Neutron Elements and vice versa, an integral piece of ElementABI specifications
* CoData -- The CoMap and CoStack data structures, named as a whole together.
* Neutron Hypervisor -- a hypervisor is what mediates between the smart contract executing within a VM and the Neutron Call System and CoData.
* Neutron ARM Hypervisor -- The specific working mechanisms of the ARM VM hypervisor will be included
* NARM VM -- The specifications for the ARM VM itself will be included here for posterity, even though it may not exactly belong here. 
* Neutron Call System -- The core interface by which all other pieces of Neutron can communicate with each other
* ElementABI -- An ABI used for communicating with Neutron Elements. It primarily uses the CoStack data structures for communication. All Neutron Element APIs must use the Element ABI.
* Neutron ABI -- The ABI used for communicating with smart contracts. Smart contracts are not forced to use it, but it is expected that the NeutronABI will be extended in backwards compatible ways, rather than a complete break from the groundwork laid down here. Using an alternative ABI would require some tooling changes, but does not require any consensus-critical changes within Neutron (ie, a hard-fork) 
* Selected Neutron Element API Standards (NEAPs) -- Neutron will include standards for various Element concepts. These include things like a standard interface for UTXO based blockchains, wallet management, cryptography standards, etc. The list described here will be non-exhaustive
* Qtum Element APIs (QEAs) -- As part of this spec, some Qtum-specific Element APIs will also be described. These are expected to expose the various unique features of Qtum which would likely not apply to other blockchains
* NeutronDB -- The consensus-critical database for smart contract state to be implemented along with Neutron. Note that Neutron could run without any database, or with an alternative one. 
* Qtum Integration -- The method by which Neutron is implemented within Qtum will be covered, at least in brief. Some parts of this may be shifted to an alternative spec due to complexity. 
* Gas Charger (name TBD) -- The component which will track gas schedules and control the costs of each Element method as well as VM operational costs. 

## CoMap

The CoMap is the central piece of communication for almost all things within Neutron. It is used for passing call data into smart contracts, for smart contracts to pass parameter data to Element APIs, and various other uses. It is expected to also be a central point of abuse for smart contracts, and so must be carefully designed to avoid exploits forming here. Each contract execution has access to three CoMap structures, titled Inputs, Results, and Outputs. The Inputs and Results CoMap is read-only. The Outputs CoMap is write-only. Is it possible to copy the entire inputs/results map into outputs, as well as to copy individual keys without involving smart contract code explicitly loading the key into memory and then storing the data into the outputs array.

Both CoMaps are quite volatile in how they operate in order to minimize memory usage and the potential for abuse. Specifically, the following happens upon entering an Element/Contract:

* The outputs CoMap from the caller is made accessible as the inputs CoMap to the callee
* A new CoMap is made for the outputs CoMap for the callee
* A new CoMap is made for the resutls CoMap for the callee

When the Element/contract returns control to the caller, the following happens:

* The outputs CoMap from the callee will overwrite the results CoMap for the caller
* The outputs CoMap from the caller is cleared and made empty
* The callee's inputs are destroyed (since they are aliased as the outputs of the caller)

Notice that over the entire contract execution, the Inputs map remains accessible and preserved, the Outputs map becomes a method by which to both return data to the calling contract, as well as to communicate with Elements or sub-contracts, and finally the Results map allows accessing sub-call results without interfering with Inputs, nor keeping every map of each sub-call accessible. In some contract call flows, it may be necessary to copy some data from a map temporarily into memory to preserve it, but this shouldn't be a common case. Ideally, as little data copying needs to be done as possible, while also putting an upper bound on the number of maps which must be preserved during contract execution. 

Note that for gas purposes, there are various pressure variables on the amount of CoMap data being tracked, thus there should be pressure to prevent using the CoMap as a generalized data store

Functions:

* push_output (key, type, data)
* push_output_with_type(key, data_with_type)
* peek_input(key) -> data
* peek_input_with_type(key) -> (type, data)
* peek_result(key) -> data
* peek_result_with_type(key) -> (type, data)
* copy_input_to_output(key)
* copy_result_to_output(key)
* count_map_swaps() -> count -- This can be used for smart data structures which can be made easily aware of invalidations to output data. This value is incremented every time the outputs map is invalidated

Note: most hypervisors are expected to include max length, beginning index, etc parameters to allow for easier subset access to each piece of data within the comaps

## CoStack

The CoStack is used for passing data in and out of ElementAPI calls. It is used rather than CoMap to avoid overhead with key names and generally unneeded complexity. There are two CoStacks available for smart contracts, read-only inputs, and write-only outputs. Smart contracts can not receive data from external contracts via the inputs stack, this can only be done by CoMaps. CoStack is only used for ElementAPI communication. Some ElementAPI functions may optionally, or by a requirement of the element, use the CoMap concept as well. 

Unlike CoMaps, there is only one instance of the two CoStacks. In other words, CoStack data is not preserved across external contract calls. The input stack will contain only the result of the external contract call and the output stack will be cleared.

Functions:

* push_stack_output(data)
* pop_stack_input(data)
* peek_stack_input(data)
* clear_stacks() -- clears both input and output stacks. Might later be rewarded with a gas refund for doing this
* pop_stack_into_output_comap()
* pop_stack_into_output_stack()

## CoStack and CoMap meta info

CoStack and CoMap data structures should be handled by the same overall interface/structure, NeutronManager

It shall be possible to get some info about the CoStack and CoMap structures

* remaining_memory() -> size
* remaining_stack_items() -> count
* reentrancy_count() -> count -- counts the number of times the current smart contract has been called within the current call stack (1 = only once)

Along with contract accessible data, there is also hidden various functions which should not be exposed to contracts directly:

* push_context(context) -> void -- this will push a new context onto the Context Tracking Stack
* peek_context(index) -> context -- this will get the context at the specified index in the stack
* pop_context() -> context -- this will destroy the current context and return it's content
* context_count() -> count -- this will return the total number of contexts currently held
* comap_released() -> bool -- (ElementAPI use only) indicates if the caller has released access to the comaps

Each context is an item in the call stack and represents an execution of a smart contract.

Constants used:

* CODATA_MAX_ELEMENT_SIZE -- proposed, 64Kb. This is the maximum size of a single stack item on the CoData. This limit has broad effects on ABI design, hypervisor implementation, and internal communications. Thus it would be extremely difficult to change after the fact.
* CODATA_MAX_TOTAL_MEMORY -- proposed, 2Mb. This is the maximum size of all elements on the stack put together, excluding context stack info. This determines the overall maximum amount of memory that can be consumed by the CoData. This would be hard coded into various pieces of infrastructure, and so reducing it after the fact will be extremely difficult. However, it can easily be made larger with minimal code changes.
* CODATA_MAX_ELEMENT_COUNT -- proposed, 256. This is the maximum number of elements that can be held on the stack. As with total memory, this is hard to make smaller after being set, but easy to make larger
* CONTEXT_STACK_MAX_COUNT -- proposed, 128. This is the total number of different contexts which can be held. It is expected that gas costs will prevent exceeding the actual count here, as each context added equates to a new VM instance and new call operation.

## Neutron Call System

This is more informally called the "Neutron Core". It is what works together with CoStack to facilitate all intercommunication between smart contracts and different ElementAPIs and thus the final underlying blockchain. 

The Call System can be called from other Elements or from smart contract code. Calling it from an external interface uses a simple interface consisting of 2 inputs (arguments) and 2 outputs (results).

Inputs:
* ElementID, u32 -- The Element to contact regarding this call
* FunctionID, u32 -- The actual function to utilize within the Element

Outputs:
* Result, u32 -- The result code. In Rust implementations, errors are handled so that it appears to give 2 outputs, one with a non-error result and one with an error result
* Non-recoverable, bool -- If this is set, then an error has occurred which means that the entire smart contract execution (including any further up contexts) must immediately terminate, reverting all state. This can happen as a result of reading out of state rent, or encountering an "impossible" error within some Neutron Infrastructure.

From this, the interface would appear quite limited, however, additional arguments, results, and data can be passed using the CoStack, which are shared between both the caller and callee. 

There is a standard for ElementIDs and FunctionIDs. Specifically, if the top bit is set on either an ElementID or FunctionID (ie, greater than 0x8000_0000) then the Element or Function is considered to be a non-standard one. This would mean that these elements/functions are not expected to be shared or implemented in any other blockchain. In addition the FunctionID of 0 is reserved. This is used to test if a particular ElementID is available. The element should return a result of 0 or greater if it exists. It should otherwise return an element does not exist error code. 

There is no standard for result codes, but anything greater than or equal to 0x8000_0000 is regarded as an error. This could include errors such as reading a stack item that doesn't exist, running out of gas, etc. In some cases this may cause the current smart contract execution to terminate, but will not cause a chain of terminations up the context stack like a non-recoverable error would. 

The other interfaces beyond actual Element calls include:

* Logging interface, for logging errors, info, debug messages, etc.
* Tracking of block height or another mechanism which is used to determine what Elements and features are currently available to smart contracts (ie, to handle forks)
* Initial loading of state data for smart contract bytecode
* Blockchain-specific methods of beginning the execution of a smart contract VM

The Call System is regarded as a "blockchain provided component". This means that each blockchain implementation will provide it's own version of the Call System. In actual implementation terms, there may be a lot of borrowed code through interfaces, traits, classes, etc. However, the Call System needs to be aware of several pieces of blockchain specific information.

The specific responsibilities of the Call System includes:

* Holding a list of ElementIDs and their mapping to Element components when called upon
* Tracking the block height and which Elements are enabled at any one time within the blockchain
* How to load smart contract bytecode from the consensus database
* How to write smart contract bytecode into the consensus database, optionally (ie, if the blockchain provides a proper writeable database)
* Initiates the top level execution into a smart contract (ie, to translate from a received transaction on the blockchain into a smart contract execution)
* Tracks the different VM hypervisors and interprets smart contract external calls to call the appropriate VM
* Implements the logging system used by other Element components for informative messages which are not tracked for consensus purposes
* Handles checkpoints (if implementing a database) within the state database to properly allow for reverting state in the case of errors

## Neutron Hypervisor

A Neutron Hypervisor is a component which exposes an interface for Neutron to a VM. Because each VM could be radically different, there is no one size fits all approach that is appropriate. However, the specific key pieces that should be exposed in a hypervisor includes:

* CoData operations, including placing data on the stack and getting data off of the stack
* Current context information, including gas used, gas limits, self-address information, etc
* An interface to call Neutron Element APIs

There is additional "typical" responsibilities such as entry point handling, memory allocation, etc. However, these are highly variable between the different types of potential smart contract VMs. Neutron is built to allow for many different types of VMs and so there are minimal demands on this specific component to accomodate this goal. 


## Built-in State

Whatever method by which Global Storage is implemented, there is a requirement for permanent (ie, not affected by rent) fields which indicate the overall characteristics and status of each smart contract deployed to the blockchain. This is standardized for the entire NeutronSystem and each hypervisor/VM must follow this standard. Specifically, any state with the prefix of `00` must be Neutron standardized data. Hypervisors should use a different prefix for internal data, bytecode, etc. 

The ContractType field will contain the following information:

* NeutronVersion: implied -- Included in the smart contract UniversalAddress; Indicates which VM etc to execute and any VM specific version info
* VMExtra: u16 -- Extra flags etc data which may be used by specific VMs/hypervisors
* Flags: u8 -- Various flags for it's execution which do not rely on the hypervisor/VM type.


The ContractType field will be held in the state key: `00 01`

The NeutronVersion field is defined as so, in Rust:

    pub struct NeutronVersion{
        pub format: u8,
        pub root_vm: u8,
        pub vm_version: u8,
        pub blockchain_version: u8
    }

Format shall always be 0 but may be given additional meaning in future versions.

root_vm shall be one of these values:

* 0 -- Unused/reserved for AAL purposes
* 1 -- Legacy EVM (TBD)
* 2 -- qx86
* 3 -- Neutron EVM (TBD)
* 4 -- WASM (TBD)

vm_version shall currently be 0, indicating "latest VM", but may be used in the future for additional features

blockchain_version shall be one of the following:

* 0 -- Unused/Reserved
* 1 -- Qtum Mainnet
* 2 -- Qtum Testnet

Note: blockchain version must match the blockchain being used as a consensus rule. ie, it is not possible to use Qtum Testnet formed addresses on Qtum Mainnet. 

Note that in the case of bare smart contract executions which are not given an address, or in the case of smart contract creation, the NeutronVersion field should be pushed before the ContractType field in the smart contract creation ABI

Proposed flags:

* Upgradeable -- Can update it's own deployed bytecode
* Stateless -- The contract can store no global storage state, aside from it's own bytecode. Noteably it can read external smart contract state and cause mutable side effects in other smart contracts by calling them
* PureContract -- Every execution of this smart contract should be assumed to be pure, with no side effects. (requires Upgradeable, Stateless, and NonPayable flag)
* NonPayable -- This smart contract should never be capable of holding coins
* (optional/Qtum only) ApprovalContract -- Only valid on bare contract executions. Indicates that the execution will opt-in for ApprovalContract behavior


Note it is not possible to modify ContractType after a smart contract has been deployed.


The ContractStatus field will be stored separately from ContractType and will contain the following information:

* Balance: u64 -- Balance in coins
* AccountingID: [u8:40] -- UTXO ID (within Qtum, may vary in other blockchains) which holds the contract's balance
* UpgradeCount: u32 -- Number of bytecode upgrades which the contract has undergone. Note this counts the number of executions which incurred one or more bytecode modifications. It does not count individual bytecode upgrades, for instance if a single execution led to 2 sections of bytecode being upgraded. Note this is Hypervisor controlled
* ABIFormat: u8 -- The specific ABI which should be used for communication with this smart contract
* ABIType: u64 -- An unverified declaration of the type of ABI this smart contract supports

The ContractStatus state will be held in the key: `00 02`

AccountingID in Qtum is composed of the following:

* TransactionIDHash: u256 -- the txid
* OutputNumber: u32 -- the output number
* ExtraInfo: u32 -- May be used in legacy EVM to indicate an AALv1 controlled transaction??


Proposed ABIFormats:

* 0 -- NeutronABI
* 1 -- Flat Ethereum ABI


Proposed ABITypes:

None yet, this will require thoughts about identifier standards.

ContractStatus can be modified, but not without restrictions. Specifically, only ABIFormat and ABIType fields may be updated and requires a hypervisor specific upgrade method. This can not be modified in stateless, pure, or non-upgradeable smart contracts

Note: In stateless blockchain models which do not incorporate a state database, all of these may be left at default values. 

Finally, the DeploymentInfo structure is data which is determined at deployment time of a smart contract and can otherwise not be modified.

* DeploymentUTXO: [u8:40] -- The UTXO ID which deployed the current smart contract code. (TBD is this needed?)
* InitialDeploymentHash: u256 -- A hash of the data which was used for the deployment of a transaction. Includes things like NeutronVersion, hypervisor specific data, and constructor input data. Used for "sub" contract deployment address generation
* DeployedFrom: UniversalShortAddress -- The responsible party or contract for this address. For "responsible party", the first spent UTXO is used as the "creator". For a smart contract, the smart contract which directly made the Element API call to create or clone this contract. 

The DeploymentInfo state will be held in the key: `00 03`

The InitialDeploymentHash uses the NeutronABI "flat" variant for constructing the hash. Specifically, it takes the entire stack at the time of smart contract construction, converts it into NeutronABI Flat variant data, and then hashes the resulting data. Thus, it is essential for consistency to ensure that extra items are not on the stack when a smart contract is created, as this will also be included into this hash. 


