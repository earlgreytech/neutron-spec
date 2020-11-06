# NeutronABI

The NeutronABI is a very simple ABI which is used to encode data to and from smart contracts as well as ElementAPIs. 

NeutronABI is an optional component for smart contracts, some may use an alternative ABI. However, all ElementAPIs must use NeutronABI. 

The ABI is very simple by design, more complex extensions to this simple design are possible but are up to interetation by smart contracts. All data is transferred via the ComStack. Each input and output is encoded as a simple item on the stack, there is no packing of multiple items onto the stack. 

The following types are defined here, following a Rust-like naming scheme:

* bool -- treated as u8, but where 0 = false and any other value is true.
* u/i8 -- single byte
* u/i16 -- 2 bytes
* u/i32 -- 4 bytes
* u/i64 -- 8 bytes
* u/i256 -- 32 bytes
* FunctionID -- 4 byte identifier
* string -- treated the same as unsized[u8]
* unsized[u8] -- variable size array of bytes
* sized[u8][n] -- fixed size array of bytes (ABI enforced to be of n size)
* unsized[u[n]] -- variable size array of items, of size n
* sized[u[n]][m] -- fixed size array of items, of size n, with a total length of m
* address -- treated the same as sized[u8][36]

The primitive types of bit size between 8 and 256 bits are simply encoded as a single stack item of the appropriate size. These items must be the exact size or smaller that is specified. If the item is shortened on the stack, then the upper bits are sign extended (ie, for signed negatives, replaced with 1 bits, otherwise and for unsigned integers it is replaced with 0 bits). Empty stack items are treated as 0. FunctionID is treated mostly the same as u32. 

unsized[u8] is treated as a single item on the stack of variable size. Thus, this item can be no larger than the ComStack element limit, 64Kb. This item also must be retrieved completely from the stack in whole, or the end truncated, with the x86 hypervisor implementation. sized[u8] is treated the same as unsized[u8], but must be exactly the size specified. 

unsized[u[n]] is more complicated as it involves multiple stack items. It begins with a simple u16 stack item indicating the size. If the size is 0, then there are no further relevant stack items. Otherwise the size is the number of items following that need to be popped from the stack. Using this mechanism within the ABI, each element within the array must be an exact size, though with the same zero extension properties of primitive integers in the case of a smaller size. The sized[u[n]][m] implementation is treated the same as the unsized variant, but does not include a u16 stack item indicating the total length, as the length is explicitly specified within the ABI. Each element given as an empty stack item is treated as an array of 0 of the appropriate "n" size.

Strings are "flat" arrays of UTF-8 character data with no preceding count of characters nor null termination. This means they are neither pascal style strings nor C style strings. Their length is known by the ComStack as the size of an item is encoded within the stack. All UTF-8 parsing of strings for display in Element APIs they should be considered lossy and best-effort. Invalid UTF-8 strings will be best-effort parsed and should not result in any smart contract visible error. Note that any consensus-visible parsing of these strings must remain consistent however to always give the same style of error handling in the case of an invalid UTF-8 string. 

This ABI can be textually represented using Rust-like function signatures. Some examples:

* `check_balance(check_address: address, tokenid: u8) -> balance: u64`
* `upload_balances(balance_list: unsized[u[44]]) -> success:bool` -- note that the u[44] here could represent a struct of form {address, u64}
* `get_data(id: u32) -> (data:unsized[u8], owner: address)`

The exact order of the stack from the perspective of these function signatures is in pop order from left to right for inputs, and pop order for the outputs. They are in otherwise from the perspective of the function being called for inputs, and the callee for outputs. 

## Examples

Given the following smart contract function: `get_data(set: u8, id: u32) -> (data: unsized[u8], owner: address)`

The expected way the function would be parsed by the called smart contract is as so:

* pop get_data functionID
* pop set u8
* pop id u32

And to construct it's set of outputs, it would then place items to the stack in the following order:

* push owner address
* push data unsized[u8]

Likewise, in order to call this function, the order of operations would be as so:

* push id u32 value
* push set u8 value
* push get_data functionID
* execute external call to contract
* pop data unsized[u8] to local variable
* pop owner address to local variable

As an example, take the more complex example: `do_calculation(input: unsized[u[32]]) -> (output: unsigned[u[u32]], error: u32)` where the input length is 2 and the output length is 3. The method for calling this function would be as so:

* push input item 2
* push input item 1
* push input length of 2
* push functionID for do_calculation
* execute external call
* pop error into local variable
* pop length of output into local variable (and use for allocation etc)
* pop output item 1
* pop output item 2
* pop output item 3

## FunctionID

FunctionID is specified as a hash of the simplified function signature.

For the function definition of `get_data(set: u8, id: u32) -> (data: unsized[u8], owner: address)`, the FunctionID would be constructed as so:

    Bottom 4 bytes of the result of `sha256(sha256("v1 get_data(u8,u32)->(unsigned[u8],address)"))`

Specifically the text is first modified:

* input and output names are removed
* spaces are removed
* output arguments will always have parentheses around them (even when just one output)
* "v1 " is added to the beginning of the argument to form an ABI version number for the function. Because spaces are otherwise illegal, this can not be user modified by intricate function names etc.
* The function name is converted to well formed UTF-8 hex data

Then finally the text is double sha256 hashed and the bottom 4 bytes of the resulting hash extracted to form the final FunctionID

## ElementAPI Variant 

The NeutronABI is used for ElementAPI calls, but with the exclusion of a FunctionID being placed on the stack, and the FunctionID being unused here. Instead a specific Element Function Number is used when doing the system_call operation, with input data being the top thing on the stack at that point. 


## Smart Contract Creation

The following items should be on the state for creating a smart contract: (in pop order)

* NeutronVersion
* ContractType 
* Hypervisor specific data...
* Constructor inputs

In the case of qx86 smart contracts:

* NeutronVersion
* ContractType
* Code section count
* Code sections...
* Data section count
* Data sections...
* Constructor inputs

## Smart Contract Bare Execution

The following items should be on the state for executing a bare smart contract: (in pop order)

* NeutronVersion
* ContractType 
* Hypervisor specific data...
* Execution inputs

In the case of qx86 smart contracts:

* NeutronVersion
* ContractType
* Code section
* Data section
* Execution inputs

# NeutronABI Flat Variant

In some cases, NeutronABI data may need to be held within a single data item or otherwise "flattened" into a simple array of bytes. This is done by using little endian encoded length prefixes (in the current case, 16 bit but if element limits were increased, it might be 32 bit) and encoding the data in push order. In this case, the following data (in pop order) would be encoded as so:

(assume inputs here are big endian encoded for simplicity)
* 0x1234
* 0x0000_0001
* 0xFF
* (empty item)
* 0x10

(note this is in byte order and little endian encoding is used for size prefixes)
    0100 -- 1 item size
    10 -- bottom element
    0000 -- 0 item size
    0100 -- 1 item size
    FF -- next element
    0400 -- 4 item size
    00000001 -- next element
    0200 -- 2 item size
    1234 -- top element

With a final encoding, with bytes encoded left to right. 

    01001000000100FF04000000000102001234



