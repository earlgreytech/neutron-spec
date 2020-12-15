# Account Access Element API

This element provides methods of accessing a smart contract's account and/or sending funds from those accounts.

ABI Information

Element ID: 9
Type: Neutron Account Tract
Provision: Recommended 

Standard Functions:

* 1, get_self_balance():static -> u64
* 2, get_balance(address: NeutronShortAddress):static -> u64
* 3, send_to(address: NeutronShortAddress, amount: u64):mutable
* 4, send_to_long(address: NeutronLongAddress, amount: u64):mutable --should be used when NeutronShortAddress can not convey enough information to send a payment to this address

# Details



