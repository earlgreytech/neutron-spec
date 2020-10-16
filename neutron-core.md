# Abstract
Neutron is a new smart contract infrastructure designed specifically for the Qtum Blockchain, but with allowances for remaining chain agnostic as well as VM agnostic. Although designed as independent components, the "stock" configuration as it will be implemented in Qtum will be described here. 

Diagram Link: https://miro.com/app/board/o9J_kjJIlZw=/


# Terms

* ComStack -- (Previously known as "SCCS") The communication stack which is used for inter-contract communication as well as used as an integral data stack which is used to pass data to and from smart contracts, to and from Neutron infrastructure pieces
* VM -- Virtual Machine, in this case, the virtual machine which executes smart contract code
* Element -- (Previously "feature sets"), an API specifically exposed to smart contracts via the ElementABI standard. Examples include "BlockchainInfo Element", "UTXO Element", "Crypto Element", etc. Some Elements defined here are to be treated as a standard, meaning that if a blockchain integrating Neutron exposes the element, it must follow the standard laid down here. 
* ABI -- Application Binary Interface, used here specifically to describe a standard method of data layout within the ComStack for communication with other smart contracts and Elements
* NeutronStar -- The API and standard set of libraries by which smart contract developers will write smart contracts which communicate with Neutron. Note that NeutronStar's user-visible design is mostly not covered in this document.

# Overall Goals

These are the informal goals of the final Qtum Neutron system. 

## Ease of Programming

* Write smart contracts in Rust and other "native" programming languages
To have allowances for porting Neutron to non-Qtum blockchains (and even non-blockchain technology). This includes allowances for "portable" smart contracts which could be deployed to other blockchains on Neutron without needing to recompile or change any code within the smart contract
* To include allowances for a "testbench" environment where smart contracts can be simulated using completely impossible or near impossible blockchain conditions, and in a significantly simpler way than typical regtest network methods
* Implement a state database system with arbitrary length keys and values, to avoid needing to do complex packing and marshaling into a specific word size.
* Initially support the x86 VM, while including allowances for later supporting a WASM VM
* Allow for smart contracts to be upgraded directly, without proxies and other complications 

## Network Security and Resource Minimization

* Implement a state database system with a built-in "black box" unobservable state rent system
* Implement "pre-compile" systems where certain smart contracts can be seamlessly converted from a live VM execution into a static pre-compile execution, without a hardfork
* Allow for smart contract infrastructure to be expanded without affecting previous smart contracts nor with a large cost to exposing such new features.
* Allow for segmented code and data sections within a single smart contract account, meaning that a smart contract could be executed without necessarily needing to load the entire code/data from the account state.
* Allow for Novel Scaling Technology
* Include infrastructure flexible enough to adopt unique and novel new VMs into Neutron
* Include "UTXO State" system, which can be utilized for transferable and one-time usable state within a UTXO of Qtum
* Include methods of interacting with and parsing UTXOs within Qtum 
* Implement an opt-in "speculative" execution mode used for determining if a smart contract transaction is eligible for being included within a block, similar to Bitcoin Script
* Implement a "simulated" execution mode where fake environments and code can be executed from within a smart contract and the results of such an execution observed

# Non-Goals

These are features or goals that will not be the target of the current specification and potentially not handled by Neutron ever.

* Increasing overall transaction capacity of Qtum (ie, increase in block size, faster blocks, etc)
* Second Layer Technology will not be designed as part of Neutron, though it is expected to cover some potential example cases to justify a Neutron feature

# Component Detail

The following components will be included within this spec:

* ComStack -- a communication stack for smart contracts talking to Neutron and vice versa, an integral piece of all ABI specifications
* Neutron Hypervisor -- a hypervisor is what mediates between the smart contract executing within a VM and the Neutron Call System and ComStack.
* Neutron x86 Hypervisor -- The specific working mechanisms of the x86 VM hypervisor will be included
* x86 VM -- The specifications for the x86 VM itself will be included here for posterity, even though it may not exactly belong here. 
* Neutron Call System -- The core interface by which all other pieces of Neutron can communicate with each other
* Neutron ABI -- The ABI used for communicating with Elements as well as other smart contracts. All Neutron Element APIs must use the Neutron ABI. Smart contracts shall use it, but it is expected that improved and alternative smart contract ABIs may arise for various needs. Using an alternative ABI would require some tooling changes, but does not require any consensus-critical changes within Neutron (ie, a hard-fork) 
* Selected Neutron Element API Standards (NEAPs) -- Neutron will include standards for various Element concepts. These include things like a standard interface for UTXO based blockchains, wallet management, cryptography standards, etc. The list described here will be non-exhaustive
* Qtum Element APIs (QEAs) -- As part of this spec, some Qtum-specific Element APIs will also be described. These are expected to expose the various unique features of Qtum which would likely not apply to other blockchains
* NeutronDB -- The consensus-critical database for smart contract state to be implemented along with Neutron. Note that Neutron could run without any database, or with an alternative one. 
* Qtum Integration -- The method by which Neutron is implemented within Qtum will be covered, at least in brief. Some parts of this may be shifted to an alternative spec due to complexity. 
* Gas Charger (name TBD) -- The component which will track gas schedules and control the costs of each Element method as well as VM operational costs. 

## ComStack

The ComStack is the central piece of communication for almost all things within Neutron. It is used for passing call data into smart contracts, for smart contracts to pass parameter data to Element APIs, and various other uses. It is expected to also be a central point of abuse for smart contracts, and so must be carefully designed to avoid exploits forming here.
In addition to being used a simple data stack for communication purposes, ComStack also implements a context tracking stack. This is not something directly accessible by smart contracts. It will track the current smart contract ID being executed, gas limit, and other information about the current execution. Note that it is possible for many different contexts to exist within a single contract executing Qtum transaction. 

Overview of all potential use cases:

* Passing call data into a smart contract either from a Qtum transaction, or from an external smart contract call
* Passing parameter data to Element APIs and receiving result data from Element APIs
* As an auxiliary storage space for some types of temporary data within smart contracts. 
* As a method of loading smart contract code and data from a Storage Element API into a Neutron Hypervisor
* As a tool of communication between Qtum and the various integration Element APIs
* As a tool of communication between different Element APIs
* Communicating with all parts of Neutron the current smart contract the gas limit it has, total gas costs, smart contract address, and other information.
* Give the wide variety of use cases, the ComStack must have strictly defined limitations and gas costs. There may be gas cost opt-out methods implemented for the ComStack for all internal uses. This is allowed to prevent the ComStack from being avoided for internal communications when it is the ideal tool for it. 

Methods of ComStack: (note, due to hypervisor and language limitations etc, this will have different actual parameters, names, etc)

* push data (data) -> void -- adds a piece of data to the top of the ComStack
* pop data () -> data -- removes the top most piece of data from the ComStack and returns it
* peek data (index) -> data -- returns a piece of data from the stack without destroying the item. In this, the top most item is 0 and the one below that is 1, etc. This allows getting various pieces of information from the ComStack without needing to do costly reordering or destruction/recreation operations
* peek data size (index) -> size -- returns the size of the data, cheaper than normal peeking
* drop data() -> void -- This will destroy the top most item on the stack without returning the data. This is cheaper than using pop due to lack of needing to copy any memory
* dup stack(index) -> void -- This is equivalent to push(peek(index)) but is cheaper due to doing the memory copying within the ComStack itself
* item count () -> count -- This will return the total number of items in the ComStack
* memory size() -> size -- This will return the total number of bytes contained within the ComStack
* push context(context) -> void -- this will push a new context onto the Context Tracking Stack
* peek context(index) -> context -- this will get the context at the specified index in the stack
* pop context() -> context -- this will destroy the current context and return it's content
* context count () -> count -- this will return the total number of contexts currently held

Limit Constants

* COMSTACK_MAX_ELEMENT_SIZE -- proposed, 64Kb. This is the maximum size of a single stack item on the comstack. This limit has broad effects on ABI design, hypervisor implementation, and internal communications. Thus it would be extremely difficult to change after the fact.
* COMSTACK_MAX_TOTAL_MEMORY -- proposed, 2Mb. This is the maximum size of all elements on the stack put together, excluding context stack info. This determines the overall maximum amount of memory that can be consumed by the ComStack. This would be hard coded into various pieces of infrastructure, and so reducing it after the fact will be extremely difficult. However, it can easily be made larger with minimal code changes.
* COMSTACK_MAX_ELEMENT_COUNT -- proposed, 256. This is the maximum number of elements that can be held on the stack. As with total memory, this is hard to make smaller after being set, but easy to make larger
* CONTEXT_STACK_MAX_COUNT -- proposed, 128. This is the total number of different contexts which can be held within the comstack. It is expected that gas costs will prevent exceeding the actual count here, as each context added equates to a new VM instance and new call operation.

## Neutron Call System

This is more informally called the "Neutron Core". It is what works together with ComStack to facilitate all intercommunication between smart contracts and different ElementAPIs and thus the final underlying blockchain. 

The Call System can be called from other Elements or from smart contract code. Calling it from an external interface uses a simple interface consisting of 2 inputs (arguments) and 2 outputs (results).

Inputs:
* ElementID, u32 -- The Element to contact regarding this call
* FunctionID, u32 -- The actual function to utilize within the Element

Outputs:
* Result, u32 -- The result code. In Rust implementations, errors are handled so that it appears to give 2 outputs, one with a non-error result and one with an error result
* Non-recoverable, bool -- If this is set, then an error has occurred which means that the entire smart contract execution (including any further up contexts) must immediately terminate, reverting all state. This can happen as a result of reading out of state rent, or encountering an "impossible" error within some Neutron Infrastructure.

From this, the interface would appear quite limited, however, additional arguments, results, and data can be passed using the ComStack, which are shared between both the caller and callee. 

There is a standard for ElementIDs and FunctionIDs. Specifically, if the top bit is set on either an ElementID or FunctionID (ie, greater than 0x8000_0000) then the Element or Function is considered to be a non-standard one. This would mean that these elements/functions are not expected to be shared or implemented in any other blockchain. 

There is no standard for result codes, but anything greater than 0x8000_0000 is regarded as an error. This could include errors such as reading a stack item that doesn't exist, running out of gas, etc. In some cases this may cause the current smart contract execution to terminate, but will not cause a chain of terminations up the context stack like a Non-recoverable error would. 

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

* ComStack operations, including placing data on the stack and getting data off of the stack
* Current context information, including gas used, gas limits, self-address information, etc
* An interface to call Neutron Element APIs

There is additional "typical" responsibilities such as entry point handling, memory allocation, etc. However, these are highly variable between the different types of potential smart contract VMs. Neutron is built to allow for many different types of VMs and so there are minimal demands on this specific component to accomodate this goal. 

## 

