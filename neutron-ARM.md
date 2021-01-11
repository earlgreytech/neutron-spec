# NARM Subset of ARM

Due to various toolchain problems as well as extra complexities that come with x86, an ARM VM is proposed as an alternative. An ARMv6-M VM has already been implemented as the "narm" project, but ARMv6-M is missing several key opcodes necessary for efficient smart contracts. Thus, ideally narm would be expanded into ARMv7 or ARMv8. 

The memory map and interface for narm is similar. Instead of interrupts being used for hypervisor communication, narm uses the `SVC` opcode. The subset would have all privileged opcodes removed, use a similar "split" of readable/writeable memory, and share a similar memory map. 

Further details on the exact subset are TBD

# The NARM hypervisor

The ARM Hypervisor ties into the the narm VM and allows for communication to and from it into Neutron. It accomplishes this by implementing a series of system calls that are exposed to smart contract programs. These system calls are simple interrupts using a consistent ABI for passing arguments etc. The hypervisor also manages the state used for a smart contract's bytecode and non-mutable data, as well as interpreting data from Neutron which can result in either a smart contract creation or call. 

## ARM Service Call ABI

This ABI will follow the "new" Linux eabi protocol. 

The 32-bit registers used for passing in data to a system call are, in order:

* r0
* r1
* r2
* r3

The registers which can be used for returning data from a system call are:

* r0
* r1 (64 bit results only)

If more data is needed, or if dynamic length data is used, then the CoStack should be used. 

For reference, given an interrupt of the format `do_stuff(foo, bar, baz, bim, fam) -> zam:u64` the register usage would be as so for input:

* r0 = foo
* r1 = bar
* r2 = baz
* r3 = bim
* first item popped from costack = fam

And as so for output:

* r0 = lower 32 bits of zam
* r1 = upper 32 bits of zam

The list of operations supported by ARM Service Calls are:

Note all operations here unless specified are classified as "pure". "variable" means that the type of operation may be pure or another type depending on exact arguments etc. 

Misc:
* SVC 0x00: nop -- Always considered a no-operation with no modifications to CPU state

CoStack operations: --note: CoStack functions are limited to 4 u32 register parameters

* SVC 0x10: push_costack (buffer: pointer, size: u32)
* SVC 0x11: pop_costack (buffer: pointer, max_size: u32) -> actual_size: u32 -- note: if buffer and max_size is 0, then the item will be popped without copying the item to memory and only the actual_size will be returned
* SVC 0x12: peek_costack (buffer: pointer, max_size: u32, index: u32) -> actual_size: u32 -- note: if buffer and max_size is 0, then this function can be used solely to read the length of the item. 
* SVC 0x13: dup_costack() -- will duplicate the top item on the stack
* SVC 0x14: costack_clear() -- Will clear the stack completely, without giving any information about what was held on the stack
* SVC 0x15: peek_partial_costack(buffer: pointer, begin: u32, max_size: u32) -> actual_amount_read: u32 -- will read only a partial amount of data from an SCCS item in the middle of the item's data (starting at 'begin')

CoMap Operations:

* SVC 0x40: push_raw_output(key: stack [u8], data_with_type: stack [u8])
* SVC 0x41: push_output(key: stack [u8], type: u32, data: stack [u8])
* SVC 0x42: peek_raw_input(key: stack [u8], max_size: u32) -> data: stack [u8]
* SVC 0x43: peek_input(key: stack [u8], max_size: u32) -> (type: u32, data: stack [u8])
* SVC 0x44: peek_raw_result(key: stack [u8], max_size: u32) -> data: stack [u8]
* SVC 0x45: peek_result_with_type(key: stack [u8], max_size: u32) -> (type: u32, data: stack [u8])
* SVC 0x46: copy_input_to_output(key: stack [u8])
* SVC 0x47: copy_result_to_output(key: stack [u8])
* SVC 0x48: get_incoming_transfer_value(address: stack NeutronAddress, id: stack u64) -> value: stack u64
* SVC 0x49: get_incoming_transfer_info(index: u32) -> (address: stack NeutronAddress, id: stack u64) -- note, both output parameters use the costack
* TBD count_map_swaps() -> count -- This can be used for smart data structures which can be made easily aware of invalidations to output data. This value is incremented every time the outputs map is invalidated


Call System Functions:

* SVC 0x20: system_call(feature, function):variable -> error:u32 -- will call into the NeutronCallSystem
* SVC 0x21: system_call_with_comap(feature, function):variable -> error:u32 -- will call into the NeutronCallSystem

CoMap operations:

* SVC 0x30: push_comap(key: [u8], abi_data: [u8], value: [u8])
* SVC 0x31: push_raw_comap(key: [u8], raw_value: [u8])
* SVC 0x32: peek_comap(key: [u8], begin: u32, max_length: u32) -> (abi_data: [u8], value: [u8]) --note max_length of 0 indicates no maximum length
* SVC 0x33: peek_raw_comap(key: [u8], begin: u32, max_length: u32) -> (raw_value: [u8])
* SVC 0x34: peek_result_comap(key: [u8], begin: u32, max_length: u32) -> (abi_data: [u8], value: [u8])
* SVC 0x35: peek_raw_result_comap(key: [u8], begin: u32, max_length: u32) -> (raw_value: [u8])
* SVC 0x36: clear_comap_key(key: [u8])
* SVC 0x37: clear_comap_outputs()
* SVC 0x38: clear_comap_inputs()
* SVC 0x39: clear_comap_results()
--todo: map copying operations

Hypervisor Functions:

* SVC 0x80: alloc_memory TBD

Context Functions:

* SVC 0x90: gas_remaining() -> limit:u64 -- Will get the total amount of gas available for the current execution
* SVC 0x91: self_address() -- result on stack as NeutronShortAddress -- Will return the current address for the execution. For a "one-time" execution, this will return a null address
* SVC 0x92: origin() -- result on stack as NeutronShortAddress -- Will return the original address which caused the current chain of executions
* SVC 0x93: origin_long() -- result on stack as NeutronLongAddress
* SVC 0x94: sender() -- result on stack as NeutronShortAddress -- Will return the address which caused the current execution (and not the entire chain)
* SVC 0x95: sender_long() -- result on stack as NeutronLongAddress
* SVC 0x96: execution_type() -> type:u32 -- The type of the current execution (see built-in types)
* SVC 0x97: execution_permissions() -> permissions:u32 -- The current permissions of the execution (see built-in types)

Contract Management Functions:

* SVC 0xA0: upgrade_code_section(id: u8, bytecode: [u8], position: u32):mutable
* SVC 0xA1: upgrade_data_section(id: u8, data: [u8], position: u32):mutable
* SVC 0xA2: upgrades_allowed(): static -> bool
* SVC 0xA4: get_data_section(id: u8, begin, max_size) -> data: [u8] --there is no code counter type provided because it can be read directly from memory. Data can as well, but may have been modified during execution


System Functions:

* SVC 0xFE: revert_execution(status) -> noreturn -- Will revert the current execution, moving up the chain of execution to return to the previous contract, and reverting all state changes which occured within the current execution
* SVC 0xFF: exit_execution(status) -> noreturn -- Will exit the current execution, moving up the chain of execution to return to the previous contract. State changes will only be committed if the entire above chain of execution also exits without any reverting operations. 

## ARM Memory Map

The ARM memory map is as follows:

* 0x10000, immutable, first code memory
* 0x20000, immutable, second code memory
* ... up to 16 code memories
* 0x80010000, mutable, first data memory
* 0x80020000, mutable, second data memory
* ... up to 16 data memories
* 0x81000000, 8Kb, mutable, stack memory (for the ARM stack)
* 0x82000000, ??? size, mutable, aux memory, loaded always as 0 and can be used as an extra RAM area

## Internal State

The data and code of a smart contract are stored separately within NeutronDB. The keys are as so:

* 0x01 00 -- first code section
* 0x01 01 -- second code section
* ... up to 16 code sections
* 0x02 00 -- first data section
* 0x02 01 -- second data section
* ... up to 16 data sections

These memory sections can be conveniently be split up and used by ELF sections in smart contract compilation which can then be parsed by a Neutron tool to remove the complexities of the ELF format, leaving only a list of memory sections with corresponding target addresses.

## Contract Creation

Contract creation is done using a specialized ABI and the ComStack. The order of elements on the comstack (in pop order) are as follows:

* VM Version info (currently unspecified)
* Section Info (specifies number of code and data sections follow)
* Code section 1
* Code section 2
* ...
* Data section 1
* Data section 2
* ...
* Contract accessible data etc follows (not used by hypervisor)

## Initial CPU State

The expected initial state when VM execution begins is all registers and flag values set to 0, excluding EIP being set to 0x10000, where execution will begin, and the following memory areas will be loaded:

* 1st code section
* stack memory accessible
* aux memory accessible

## Loading Memory Areas

Each code section and data section which is attempted to be accessed by the VM will result in loading that section from NeutronDB without any smart contract visible error (unless the section does not exist). This will incur a memory size gas cost as well as the gas cost for loading the state from NeutronDB. There is no explicit operation to load or unload memory and it is instead done implicitly by trying to access that memory

Note that in the case of one-time executions, no state will be stored in NeutronDB for the contract, nor will any state be loaded (except by external contract calls and external state loads) from NeutronDB. Instead, the entire set of code and data memory data will be stored within the transaction data and loaded into the Hypervisor via the ComStack. 

## Built-in Types

The following definitions (in Rust) are used for NeutronShortAddress and NeutronFullAddress

    #[repr(C)]
    pub struct NeutronShortAddress{
        pub version: u32,
        pub data: [u8; 20]
    }

    #[repr(C)]
    pub struct NeutronFullAddress<'a>{
        pub version: u32,
        pub data: &'a [u8]
    }

NeutronShortaddress is a fixed size, 24 byte structure which can be used for all address uniqueness and identification purposes, however, it can not be used for sending Qtum to an address EXCLUDING perfect-conversion addresses. Perfect-conversion addresses are address types which can be converted perfectly from a short address to a full address. In otherwords, the entire address can be contained within 24 bytes. NeutronFullAddress is a dynamic length structure which contain an address of any size and can be used for sending Qtum. For simplicity and lower resource usage, the short form of an address should be used in all places where possible. It is possible to convert a full address to a short address, but not always the other way around. 

The following constants are also defined:

    EXECUTION_TYPE_CALL     = 0
    EXECUTION_TYPE_DEPLOY   = 1
    EXECUTION_TYPE_ONE_TIME = 2

* CALL -- A call from a transaction or from an external contract into an existing smart contract
* DEPLOY -- A new smart contract is being deployed
* ONE_TIME -- A piece of smart contract code is being executed which has not and will not be saved to the blockchain permanently and will not be assigned an account/address 

Execution Permissions can be one or more of the following flags:

* Mutable call -- standard mutable call with no restrictions
* static call -- A static call which can read external contract and otherwise mutable internal data, but can not modify any data or make mutable calls
* Pure call -- a restricted pure call which can only read immutable internal data and can not otherwise access any external data and can not modify any data or make any mutable or static calls.

Defined as so:

    EXECUTION_PERMS_MUTABLE     = 1
    EXECUTION_PERMS_STATIC      = 2
    EXECUTION_PERMS_PURE        = 4

All smart contract APIs for checking this, should be a bitwise comparison:

    if (permissions & EXECUTION_PERMS_MUTABLE) > 0
        //capable of doing everything a mutable call can
    if (permissions & EXECUTION_PERMS_STATIC) > 0
        //capable of doing everything a static call can
    if (permissions & EXECUTION_PERMS_PURE) > 0
        //capable of doing everything a pure call can

Furthermore, in order to express that an execution is mutable (which would incldue the permissions for static and pure) it should be written as so:

    permissions = EXECUTION_PERMS_MUTABLE + EXECUTION_PERMS_STATIC + EXECUTION_PERMS_PURE

All unspecified bits are reserved for future additions to these permissions. In the case of a permission being created that is even more restrictive than "pure", then none of these flags should be set. In the case of a permission being created which gives more power than "mutable", then all of these flags should be set, along with a new flag conveying this new permission.

## Hypervisor Internal State

All narm internal state has a prefix of `02`. It specifically stores the following state:

* `0200` - `020F` -- code section state
* `0210` - `021F` -- data section state

Note that this state is affected by state rent and is restored via the typical methods and persisted by actually using the code/data sections within a smart contract execution.


