In this folder all of the various elements will be described


Element Function Modifiers:

These are only used in these specifications and are not necessarily strict. 

* :const -- Conveys that it will express data which may change in later blocks or transactions, but will be constant across the entire current chain of execution. Introduces no current execution data dependencies, but does introduce whole-blockchain data dependencies. Thus these can not be used by pure execution contexts. These types of functions are effectively less dynamic than static, but more dynamic than pure. 
* :pure -- Conveys that the function is purely computation and thus given the same input should give the same output at any point in the blockchain history. Can be used by all execution contexts (short of forks etc for adding new algorithms or bug fixes)
* :static -- Conveys that the function will not modify or alter potentially upstream data dependencies, but also that this function's results may change through the current chain of execution. In other words, it may read execution-mutable state, but not modify it. This can not be used by pure execution contexts
* :mutable -- Conveys that the function will modify data which may be relied upon in other places in the execution history. In other words, it may both read and write to execution-mutable state. This can not be used by pure nor static execution contexts. 

Note that these modifiers are not included in the FunctionID

Every Element API must implement the following function:

* 0, function_exists(function_id: u32) -> version: u32

If version in this case is 0, then the function is not implemented. Any other value means it is implemented. Numbers other than 1 may be used to indicate additional information that is implementation-defined

This function shall be used rather than speculative execution (ie, pushing inputs and then checking if the error indicates the function did not exist) to save on gas costs and also because an unimplemented function will mean that the stack is returned in a dirty state, as the Element API does not understand how to remove only the given inputs. 

Calling any function_exists Element API function shall be a free function in terms of gas costs. C

Calling function_exists on an element which is not supported will not return a 0 version, but rather give an error which will indicate the element is not supported. Usage of the API for this purpose to detect supported Elements shall also be gas-free. 

All ABI parameters are listed here as normal function signatures like in Rust, but keep in mind that with CoMap there is not an explicit order for parameters. The order listed here will be used with the official Neutron Star Rust API. With other APIs there may be modifications to this order, or usage of structs for parameters, etc. When there are optional parameters, the Rust concept of `Option<T>` is used in the Neutron Star Rust API.

The types are specified here for convenience, but are not included in the CoMap structure, with the exclusion of external contract calls

Standard Tracts:

* Neutron Standard -- These are standard Neutron functions which should not rely on any particular blockchain design, though not all of these are mandatory
* Neutron Account Tract -- These are Neutron functions which rely on an account based blockchain model, or an account abstracted model (as implemented in Qtum)
* Neutron UTXO Tract -- These are Neutron functions which rely on a UTXO based blockchain model

Provisions:

* Mandatory -- All implementations of Neutron must implement these functions
* Recommended -- This is recommended functionality and typically lightweight things to adapt for most blockchains
* Optional -- These tend to depend on blockchain design or may be more involved or with less usefulness, and thus may be excluded from some blockchain implementations of Neutron