# External Contract Element API

This element provides methods of interacting with external smart contracts stored on the blockchain.

ABI Information

Element ID: 3
Type: Neutron Standard
Provision: Recommended if underlying blockchain has the capability

Standard Functions:

* 1, call_contract(address: NeutronShortAddress, gas_limit: u64, inputs: ...):mutable -> (error_code:u32, outputs: ...)
* 3, static_call_contract(address: NeutronShortAddress, gas_limit: u64, inputs: ...):static -> (error_code:u32, outputs: ...)
* 4, pure_call_contract(address: NeutronShortAddress, gas_limit: u64, inputs: ...):pure -> (error_code:u32, outputs: ...)

Optional Functions:

* 2, call_contract_with_value(address: NeutronShortAddress, gas_limit: u64, value: u64, inputs: ...):mutable -> (error_code:u32, outputs: ...)

## Details

### Types of calls

A static call is one in which no state can be written to the blockchain. A pure call is one in which no additional state can be loaded beyond the contract's internal bytecode state needed for execution. A pure call could eventually be optimized in certain ways as it would have no state dependencies and thus may result in discounted gas fees.

A static call execution strictly comes with the following restrictions:

* Can only make use of static, pure, and const Element API Functions

A pure call execution comes with the following restrictions:

* Can only make use of pure Element API Functions


