# Context Storage Element API

This element provides a basic method of accessing a context storage mechanism for non-persistent data that can be shared across smart contract executions

ABI Information

Element ID: 10
Type: Neutron Standard
Provision: Recommended

Standard Functions:

* 1, load_state(key: [u8]):static -> value:[u8]
* 2, store_state(key: [u8], value: [u8]):mutable

Standard Optional Functions:

* 3, key_exists(key: [u8]):static -> exists:bool
* 4, load_external_state(address: NeutronShortAddress, key: [u8]):static -> value:[u8]

Note that NeutronDB as it is currently designed would not implement the `key_exists` function

## Details

This is for sharing storage across a single "context". In Qtum, this would be a single overall block. Data stored within this structure is not persisted after the block is done being processed.
