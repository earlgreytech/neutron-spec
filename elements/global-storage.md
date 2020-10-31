# Global Storage Element API

This element provides a basic method of accessing a global storage mechanism for persistent data across smart contract executions

ABI Information

Element ID: 2
Type: Neutron Standard
Provision: Recommended if the underlying blockchain is capable of persistent data

Standard Functions:

* 1, load_state(key: [u8]):static -> value:[u8]
* 2, store_state(key: [u8], value: [u8]):mutable

Standard Optional Functions:

* 3, key_exists(key: [u8]):static -> exists:bool
* 4, load_external_state(address: NeutronShortAddress, key: [u8]):static -> value:[u8]

Note that NeutronDB as it is currently designed would not implement the `key_exists` function

## Details

