# Contract Creation Element API

This element provides the ability for smart contracts to create other smart contracts. In general this gives the smart contract creator no special abilities over the contract created, unless specifically coded to include some administrator functionality to the creator of the contract. 

ABI Information

Element ID: 7
Type: Neutron Standard
Provision: Recommended 

Standard Functions:

* 1, create(gas_limit: u64, vm: NeutronVersion, type: ContractType, hypervisor_inputs: ..., constructor_inputs: ...) ->  (error_code:u32, address: NeutronShortAddress, outputs: ...)
* 2, create_deterministic_from_new_bytecode(gas_limit: u64, salt: u256, vm: NeutronVersion, type: ContractType, hypervisor_inputs: ..., constructor_inputs: ...) ->  (error_code:u32, address: NeutronShortAddress, outputs: ...)
* 3, create_deterministic_from_sender_bytecode(gas_limit: u64, salt: u256, vm: NeutronVersion, type: ContractType, hypervisor_inputs: ..., constructor_inputs: ...) ->  (error_code:u32, address: NeutronShortAddress, outputs: ...)

Optional Functions:

* 4, clone_smart_contract(gas_limit: u64, clonee: NeutronShortAddress, constructor_inputs: ...) -> (error_code:u32, address: NeutronShortAddress, outputs: ...)
* 5, clone_smart_contract_deterministic_from_new_bytecode(gas_limit: u64, salt: u256, clonee: NeutronShortAddress, constructor_inputs: ...) -> (error_code:u32, address: NeutronShortAddress, outputs: ...)
* 6, clone_smart_contract_deterministic_from_sender_bytecode(gas_limit: u64, salt: u256, clonee: NeutronShortAddress, constructor_inputs: ...) -> (error_code:u32, address: NeutronShortAddress, outputs: ...)


## Details

Each of the different types of creation calls  behave the same except in the way that the new contract address is determined. 

Note that in this case "bytecode" means everything that was on the stack when the new smart contract deployment began, including constructor inputs, NeutronVersion, bytecode, etc fields. 

Creation from Transaction (for reference):

    hash(0x00 ++ DeploymentTxID ++ OutputNumber)

Create algorithm:

    hash(0x01 ++ ExecutingTxID ++ OutputNumber ++ CreationCounter)

Deterministic From New Bytecode:

    hash(0x02 ++ ExecutingAddress ++ Salt ++ NewBytecodeHash) 

Deterministic from Sender bytecode:

    hash(0x03 ++ ExecutingAddress ++ Salt ++ SenderBytecodeHash) -- note: originally deployed bytecode is used here for the hash. It will not take into account bytecode upgrades


