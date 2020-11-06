# External Contract Element API

This element provides methods of interacting with external smart contracts stored on the blockchain.

ABI Information

Element ID: 3
Type: Neutron Standard
Provision: Recommended if underlying blockchain has the capability

Standard Functions:

* 1, call_contract(address: NeutronShortAddress, gas_limit: u64, inputs: ...):mutable -> (error_code:u32, outputs: ...)
* 2, static_call_contract(address: NeutronShortAddress, gas_limit: u64, inputs: ...):static -> (error_code:u32, outputs: ...)
* 3, pure_call_contract(address: NeutronShortAddress, gas_limit: u64, inputs: ...):pure -> (error_code:u32, outputs: ...)
* 4, get_self_address_of_deployer():pure -> address: NeutronShortAddress
* 5, get_address_of_deployer(address: NeutronShortAddress):static -> address: NeutronShortAddress
* 6, get_self_initial_deploy_hash():pure -> hash: u256
* 7, get_initial_deploy_hash(address: UniversalShortAddress):static -> hash: u256
* 8, get_self_contract_flags():pure -> flags: u8
* 9, get_contract_flags(address: UniversalShortAddress):static -> flags: u8
* 10, get_self_upgrade_count():const -> count:u32
* 11, get_upgrade_count(address: UniversalShortAddress):static -> count:u32
* 12, get_self_abi_format():const -> format: u8
* 13, get_self_abi_type():const -> type: u64
* 14, get_abi_format(address: NeutronShortAddress):static -> format: u8
* 15, get_abi_type(address: NeutronShortAddress):static -> type: u64
* 
Optional Functions:

* 16, call_contract_with_value(address: NeutronShortAddress, gas_limit: u64, value: u64, inputs: ...):mutable -> (error_code:u32, outputs: ...)

## Details

### Types of calls

A static call is one in which no state can be written to the blockchain. A pure call is one in which no additional state can be loaded beyond the contract's internal bytecode state needed for execution. A pure call could eventually be optimized in certain ways as it would have no state dependencies and thus may result in discounted gas fees.

A static call execution strictly comes with the following restrictions:

* Can only make use of static, pure, and const Element API Functions

A pure call execution comes with the following restrictions:

* Can only make use of pure Element API Functions


