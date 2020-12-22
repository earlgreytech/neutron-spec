# NeutronABI

The NeutronABI is an ABI which is used to encode data to and from smart contracts as well as ElementAPIs. 

NeutronABI is an optional component for smart contract communication, some may use an alternative ABI. However, all ElementAPIs must use NeutronABI. 

The ABI is simple, yet powerful by design with just enough complexity to faciliate ample opportunity for extension. The map concept that is native to Neutron is codeified into a standardized format for key names:

The name format is as follows:

    .[[@/~][number]namespace].[@/~][number]keyname -> [type]data

Type is one of the following values:

* 0 -- u8
* 1 -- i8
* 2 -- u16
* 3 -- i16
* 4 -- u32
* 5 -- i32
* 6 -- u64
* 7 -- i64
* 8 -- u128
* 9 -- i128
* 10 -- u256
* 11 -- i256
* 12 -- u512
* 13 -- i512
* 14 -- u1024
* 15 -- i1024
* 16-31 -- reserved
* 32 -- [u8]
* 33 -- [i8]
* 34 -- [u16]
* 35 -- [i16]
* 36 -- [u32]
* 37 -- [i32]
* 38 -- [u64]
* 39 -- [i64]
* 40 -- [u128]
* 41 -- [i128]
* 42 -- [u256]
* 43 -- [i256]
* 44-63 reserved
* 64 -- NeutronAddress
* 65 -- NeutronLongAddress
* 66 -- NeutronVersion
* 67 -- ASCII string
* 68 -- UTF-8 string
* 69 -- Function ID 
* 70 -- big endian u256 (for compatibility with ETH displaying of u256 values)
* 71 -- big endian u160
* 72 -- Bitcoin style address (will display as base58)
* 73 -- big endian u512
* 74 -- big endian u1024
* 75 -- big endian u2048
* 128 - 195 -- Extension types (can be defined by community/users), treated as [u8] if not implemented
* 196-227 -- One byte extension types (can be defined by community/users). Following byte is used for more type information
* 228-243 -- Two byte extension types (can be defined by community/users). Following two bytes are used for more type information
* 244-252 -- Four byte extension types (can be defined by community/userS). Following four bytes are used for more type information
* 253 -- Reserved
* 254 -- Variable length extension. Followed by a single byte length field, and then the following bytes up to that length encodes the full type information.
* 255 -- Unknown/Generic


The namespace portion of the key is `.` for the "root" namespace. The `..` namespace is reserved for system use. Such keys can not be written by smart contracts. Otherwise, every namespace must begin and end with `.`. The namespace concept has no fixed use case, but has significant potential to improve the extensibility of ABIs. Examples:

* Versioning of ABI calls can be done, such as by specifying `.v2.`. This allows the same function to be used for both v1 and v2 parameters and behavior
* Custom extensions to existing functions, such as by specifying the contract's name `.mydapp.`. This allows for a standardized function to be extended with new or unique behavior for callers that are aware of these extensions
* Anti-passthrough parameters passed to other functions. For example, if a contract is called by a user who intends to call another contract down the line, a namespace like `.rootContract.` could be used, with this root contract interpreting the parameters as needed, and stripping them off of the map before passing-through to the next contract call. 
* Structures are easily built up like `c.prefix_info.character` and `a.prefix_info.owner` where both owner and character belong to the prefix_info "structure". 

With the CoMap concept combined with this ABI, it is trivial to not only send multiple pieces of data as parameters into calling contracts, but to also return multiple pieces of data as results from the called contract. The same ABI rules apply. 

All integer types unless indicated are displayed as little endian numbers

## Key encoding

All keys are treated strictly as ASCII strings, and displayed accordingly in UI etc. The "z/Z" prefix is used to indicate a key name forms an array. In order to prevent the complexities of BCD from being required, this is the one exception to ASCII key namees.

Every key part is prefixed by `.`, if the next character is either `@` or `~` then it indicates that it forms an array. The character after the `@` or `~` is treated as either an unsigned byte, or unsigned word, respectively, to indicate an array element.

There is no searching across keys available in the CoMap concept, so there should be a count as well. This is represented by using the character `#` as the prefix of the key. The data in the key should be either u8 or u16 typed. If there are less array elements then this number indicates, then it can be assumed the missing elements are null (ie, Option::None in Rust terms)


## Reserved Keys And Guidelines

* Every key that has a `$` namespace is for system use only. It may be stripped or otherwise modified and can not be read from smart contracts, but can be written from smart contracts
* The key `.*fn` is the function ID which is to be called. It is allowed, but not defined/implemented to have multiple function calls like `f.1`, `f.2` etc
* The key "c.abi" is used for custom extensions or where ABI versioning is required. This has no impact on NeutronABI implementations, but is a specifically reserved key
* A key prefix of `!` indicates a Neutron specific input. It can be written to by smart contracts, but can not be read from smart contracts. It's exact data may be modified in transit etc
* A key prefix of `?` indicates a Neutron specific output. It can be read from a smart contract, but can not be written to. It's exact data is determined by the Neutron subsystems
* A key prefix of `%` indicates a reserved Neutron value. Such keys can not be read nor written to by smart contracts
* All user keys should start with `.`, this may be a consensus enforced restriction
* Namespaces can be nested. This can be used for namespaces as well as for forming simple loosely typed structures. For instance, this key name is valid `.v2.@[0]SendingData.Form`. This would be the struct "sending data" (as the 0th element of an array), located in the v2 namespace, and with the key name of "form". 
* It is not enforced by consensus, but the following characters should not be used in user key names. This may or may not be enforced by consensus:
    * "
    * '
    * []
    * !
    * $
    * %
    * ^
    * `*` -- note: can only be used for the `.*fn` function indicator
    * ()
    * /
    * |
    * \
    * ;
    * <>
    * ,
    * `
    * ?
* It is not always necessary to include a `#` count input for an array. If the array is contiguous with no missing elements, then it can simply be iterated over starting from 0, and will assume that the first missing item found indicates the end of the array.

## FunctionID

FunctionID is more simply an interface name and a function name of the following format:

    [interface].function

Thus, non-interface functions should be prefixed with `.`

FunctionIDs can be used by the ABI Helper element for easier mapping that does the string comparisons in native code??

## ElementAPI Variant 

To simplify ElementAPI, there is a simplified read-only "input" and write-only "output" stack. Similar to CoMap, output of the caller is aliased to the input of the callee, while upon returning, the output of the callee is moved to the input of the caller while the outputs of the caller is cleared. Unlike CoMap, stack manipulation tools available are more limited by design. The CoMap can also be used, but must be explicitly specificied in the Element API call. For instance, there might be `element_call` and `element_call_with_comap`. Each ABI function should specify if the comap version is required. 

If the comap call is used, then the results comap will be cleared and made to match the outputs of the Element function, and the outputs comap will be cleared. Even if the Element function does not use the comap, this behavior will still be used when the comap call is used. Some functions may have extra features available by using comap, or include extensions within comap data. Certain functions may require comap data.

If a comap ElementAPI is used, then it should include appropriate type info, preferably a single byte type data.

# Flat and Semi-flat ABI Encoding

In certain cases, such as in transaction data, there may not be specific support for map style data to be stored, and thus it is necessary to encode this data using a flat or semi-flat ABI. Where flat would be a single contiguous stream of bytes, and semi-flat would be an array of byte streams (such as a stack). In Qtum specifically, the flat method will be used for encoding such ABI data within Qtum transactions.

The format of such data is as so:

* length of key name
* key name (which includes namespace, array number, etc)
* length of data
* data (including type data)

This flat data will be decoded by Neutron and put into the CoMap structure


## Smart Contract Creation

The following items should be contained in the CoMap:

* NeutronVersion
* ContractType 
* Hypervisor specific data...
* Constructor inputs

In the case of ARM contracts:

* `!.v` -> NeutronVersion
* `!.t` -> ContractType
* `!.c` -> primary code section
* `!.d` -> primary data section
* `!.#extra_code` -> Extra code section count
* `!.@extra_code` -> Extra Code sections...
* `!.#extra_data` -> Extra data section count
* `!.@extra_data` -> Extra Data sections...
* Constructor inputs

If #extra_code and/or #extra_data is missing, then it is assumed that there is no such sections

## Smart Contract Bare Execution

The following items should be on the state for executing a bare smart contract: (in pop order)

* NeutronVersion
* ContractType 
* Hypervisor specific data...
* Execution inputs

In the case of qx86 smart contracts:

* `!.v` -> NeutronVersion
* `!.t` -> ContractType
* `!.c` -> Code section (note, only one section of each data and code are valid)
* `!.d` -> Data section
* Execution inputs

# NeutronABI Function Modifiers

Similar to the ElementAPI specifications, it is possible to postfix a function modifier to indicate the type of permissions needed for the function. Specific modifiers include:

* static
* pure (assumes nonpayable)
* payable
* nonpayable
* mutable (the default if no modifier is present, nonpayable assumed)

Examples:

    do_computation:pure
    do_something:static,payable

Note that modifiers are not used when computing function IDs and thus are only for the sake of ABI code generation and programmer reference.
