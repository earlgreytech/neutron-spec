# NeutronABI

The NeutronABI is an ABI which is used to encode data to and from smart contracts as well as ElementAPIs. 

NeutronABI is an optional component for smart contract communication, some may use an alternative ABI. However, all ElementAPIs must use NeutronABI. 

The ABI is simple, yet powerful by design with just enough complexity to faciliate ample opportunity for extension. The map concept that is native to Neutron is codeified into a standardized format for key names:

The name format is as follows:

    type[namespace]keyname -> data

Type is one of the following values:

* f -- function ID (see reserved keys)
* b -- bool (0 = false, all other values are true)
* c -- u8
* s -- u16
* i -- u32
* l -- u64
* z -- u256
* h -- h256, 256bit hash (u256 alias)
* a -- address
* v -- [u8]
* C -- i8
* S -- i16
* I -- i32
* L -- i64
* s -- string ([u8] alias)
* u -- UTF-8 string
* x -- extension (followed by one or more type indication characters, treated as [u8] otherwise)
* X -- Neutron specific data extension (encodes some commonly used structures)
* . -- reserved for system use
* @? -- [short][type], used for unnamed keys (such as pieces of an array), with the type name following the !, such as `@z` to create an array of hashes, like `@?z.hashes` with a numerical, non-ASCII byte added after the `@` character (replacing the ? character)
* ~? -- [long][type], used for unnamed keys in the same way as `@` but where the number of keys could exceed 256 and thus a 16 bit integer should be used for the counting. Note the counting number is treated as a little endian u16 type.
* any other character is treated as [u8]


The namespace portion of the key is `.` for the "root" namespace. The `..` namespace is reserved for system use. Such keys can not be written by smart contracts. Otherwise, every namespace must begin and end with `.`. The namespace concept has no fixed use case, but has significant potential to improve the extensibility of ABIs. Examples:

* Versioning of ABI calls can be done, such as by specifying `.v2.`. This allows the same function to be used for both v1 and v2 parameters and behavior
* Custom extensions to existing functions, such as by specifying the contract's name `.mydapp.`. This allows for a standardized function to be extended with new or unique behavior for callers that are aware of these extensions
* Anti-passthrough parameters passed to other functions. For example, if a contract is called by a user who intends to call another contract down the line, a namespace like `.rootContract.` could be used, with this root contract interpreting the parameters as needed, and stripping them off of the map before passing-through to the next contract call. 
* Structures are easily built up like `c.prefix_info.character` and `a.prefix_info.owner` where both owner and character belong to the prefix_info "structure". 

With the CoMap concept combined with this ABI, it is trivial to not only send multiple pieces of data as parameters into calling contracts, but to also return multiple pieces of data as results from the called contract. The same ABI rules apply. 

## Key encoding

All keys and string data are treated strictly as ASCII strings, and displayed accordingly in UI etc. UTF-8 string data is not validated and is "best attempt" decoded for display. For display purposes, all type prefixes should be removed if possible before displaying the key name.

The only exception to the ASCII-only policy in key names is when using the `@` or `~` type variant is used. With these types, the 2nd (and 3rd in the case of `~`) byte are removed along with the type prefix before display. Ideally, the display represents an element like `@?z.structure.hash` as so:

* [0]structure.hash 
* [1]structure.hash
* etc



## Reserved Keys And Guidelines

* Every key that has a prefix of "." has a reserved system use and can be read from smart contracts, but can not be written to
* No key shall have a prefix of `[`. In such a case, the key would be rejected from being added
* Every key that has a `$` namespace is for system use only. It may be stripped or otherwise modified and can not be read from smart contracts, but can be written from smart contracts
* The key `f.` is the function ID which is to be called. It is allowed, but not defined/implemented to have multiple function calls like `f.1`, `f.2` etc
* The key "c.abi" is used for custom extensions or where ABI versioning is required. This has no impact on NeutronABI implementations, but is a specifically reserved key
* Keys which begin with ! are reserved for unnamed keys
* It is not enforced by consensus, but the following characters should not be used in key names:
    * "
    * '
    * []
    * #
    * `
* All structures are displayed in a "flat" way. Thus, it's not possible to have something like `structure[2].sub[10].data`. Instead, such a thing would be displayed like `[20]structure.sub.data`. If absolute sub structure arrays like this are required, then the key should name it explicitly `structure.2.sub.10.data`, though this comes with significant impacts in how keys can be scanned efficiently, and ASCII integers should be used
* If empty array elements are desired, then `.count` should be suffixed to the name of an array and used as the key. There is otherwise no method of searching across the map via wildcard or other methods

## FunctionID

FunctionID is more simply an interface name and a function name of the following format:

    [interface].function

Thus, non-interface functions should be prefixed with `.`

FunctionIDs can be used by the ABI Helper element for easier mapping that does the string comparisons in native code??

## ElementAPI Variant 

The NeutronABI is used for ElementAPI calls, but with the exclusion of a FunctionID being placed in the map. Instead a specific Element Function Number is used when doing the system_call operation, and with CoMap being used for all parameter data

## Unnamed keys

It is possible to emulate a stack for easier usage of variable numbers of parameters. Specifically, these unnamed keys can only be accessed via push, pop, and peek. 


# Flat and Semi-flat ABI Encoding

In certain cases, such as in transaction data, there may not be specific support for map style data to be stored, and thus it is necessary to encode this data using a flat or semi-flat ABI. Where flat would be a single contiguous stream of bytes, and semi-flat would be an array of byte streams (such as a stack). In Qtum specifically, the flat method will be used for encoding such ABI data within Qtum transactions.

The format of such data is as so:

* length of key name
* key name (which includes type name, namespace, etc)
* length of data
* data

This flat data will be decoded by Neutron and put into the ABI mappings


## Smart Contract Creation

The following items should be contained in the CoMap:

* NeutronVersion
* ContractType 
* Hypervisor specific data...
* Constructor inputs

In the case of ARM contracts:

* `Xv$` -> NeutronVersion
* `Xt$` -> ContractType
* `v$code` -> primary code section
* `v$data` -> primary data section
* `@?v$code` -> Extra Code sections...
* `@?v$data` -> Extra Data sections...
* Constructor inputs

## Smart Contract Bare Execution

The following items should be on the state for executing a bare smart contract: (in pop order)

* NeutronVersion
* ContractType 
* Hypervisor specific data...
* Execution inputs

In the case of qx86 smart contracts:

* `Xv$` -> NeutronVersion
* `Xt$` -> ContractType
* `v$code` -> Code section (note, only one section of each data and code are valid)
* `v$data` -> Data section
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
