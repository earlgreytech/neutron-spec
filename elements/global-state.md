# Global State Element API

This element provides a basic method of accessing a global storage mechanism for persistent data across smart contract executions

ABI Information

Element ID: 2
Type: Neutron Standard
Provision: Recommended if the underlying blockchain is capable of persistent data

Standard Functions:

* 1, load_state(key: [u8]):static -> value:[u8]
* 2, store_state(key: [u8], value: [u8]):mutable

Standard Token Functions:

* X, set_token_balance(id: u32, address: NeutronShortAddress, value: u32); -- only usable by the token owner
* X, get_token_balance(token_owner: NeutronShortAddress, id: u32, address: NeutronShortAddress):mutable -> value: u64
* X, send_token_balance(token_owner: NeutronShortAddress,, id: u32, to: NeutronShortAddress, value: u64):mutable -> new_balance: u64
* X, get_token_permissions(token_owner: NeutronShortAddress, id: u32):static -> permissions: u64
* X, set_token_permissions(token_owner: NeutronShortAddress, id: u32, permissions: u8):mutable

Standard Optional Functions:

* 3, key_exists(key: [u8]):static -> exists:bool
* 4, load_external_state(address: NeutronShortAddress, key: [u8]):static -> value:[u8]
* 5, store_input_comap(prefix: [u8], ...):comap, mutable -- stores every comap value into storage, preserving type info with the data, and lists the data under the given keynames in the comap, with the given prefix attached to each key name
* 6, store_result_comap(prefix: [u8], ...):comap, mutable -- same as above
* 7, load_comap(prefix: [u8], ...):comap, static -> (...) -- loads all of the storage keys listed in the caller's output comap (data is ignored), with the prefix given prefixing each key name, into the caller's result comap 
* X, send_long_token_balance(token_owner: NeutronShortaddress, id: u32, to: NeutronFullAddress, value: u64):mutable -> new_balance: u64



Note that NeutronDB as it is currently designed would not implement the `key_exists` function

## Details

Token state is treated internally as the same concept as regular load/store state, but with special rules applied for setting the value. send_token_balance is equivalent to the following for reference:

    assert!(get_token_permissions(id) & TOKEN_PERMISSION_FREELY_TRANSFER > 0);
    let from_balance = get_token_balance(owner, id, self_address);
    let to_balance = get_token_balance(owner, id, to);
    assert!(value <= from_balance);
    to_balance += value;
    from_balance -= value;
    let to_key = generate_private_token_key(id, to);
    private_store_state_as(owner, to_key, to_balance); //sets private state within the token owner 
    let from_key = generate_private_token_key(id, from);
    private_store_state_as(owner, from_key, from_balance);

Token permission flags:

* Freely transferrable (can be transferred without invoking the owner)

By expressing token state in this way, it is possible to use a "built-in" address for communicating with an external blockchain, ie, Qtum, so that an AAL etc can emit proper transaction data onto the underlying blockchain


