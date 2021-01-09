# Extended Address Element API

This element provides methods for registering and thus interacting with "extended address" forms of a NeutronAddress

ABI Information

Element ID: XX
Type: Neutron Optional
Provision: Only required if needed by the underlying platform

Mandatory Functions:

* 1, get_address_data(address: NeutronAddress) -> data: [u8]

Recommended Functions:

* 2, register_address(data: [u8]) -> address: NeutronAddress

Extension Functions for multi-addresses:

* 3, get_multi_addresses(address: NeutronAddress) -> (count: u32, [address: NeutronAddress]) -- should return a recoverable error if address is not a multi-address
* 4, register_multi_address([address: NeutronAddress]) -> address: NeutronAddress

Extension Functions for Bitcoin Forks:

* 5, decode_address(address: NeutronAddress) -> [bitcoin_ops: variable] -- If implemented, must work on extended addreses. Can optionally also decode other address versions

## Details

The version number used by generated address values is `0x8000_0001`. Generated NeutronAddress should use a hash of the data, or otherwise the same data should generate the same address. Blockchains which use extended addresses as potential inputs into smart contracts should implicitly use register_address for such addresses before contract execution begins. In this way, if sender_address is an extended address, then it should be possible to directly use get_address_data on this address without any registration. 

The data which is held by such extended addresses is completely platform defined and as such data modifications to such extended addresses should be regarded as never platform independent.

The register_multi_address call should generate it's resulting NeutronAddress by hashing the addresses given in an implicitly sorted order. This is so that even if addresses are specified in a different order, the same address would be produced. Blockchains exposing multi-sig addresses to smart contracts via items such as sender_address should also use this sorting mechanism. Note that with pay-to-scripthash addresses it's possible to construct a "blind" multi-sig address where the keys are not known and all needed data for validation is contained within a single hash. These addresses would not be represented by an extended address. 

Extended addresses can hold token balances and potentially even hold state, but can not natively be called or executed like normal smart contract addresses.


