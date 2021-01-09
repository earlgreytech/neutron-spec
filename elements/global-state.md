# Global State Element API

This element provides a basic method of accessing a global storage mechanism for persistent data across smart contract executions

ABI Information

Element ID: 2
Type: Neutron Standard
Provision: Absolutely required for Neutron to operate

Mandatory Functions:

* 1, load_state(key: [u8]):static -> value:[u8]
* 3, key_exists(key: [u8]):static -> exists:bool

Mandatory Token Functions:

Note: if there is no support for any type of tokens in the underlying platform, these should always return 0 

* X, get_token_balance(token_owner: NeutronShortAddress, id: u32, address: NeutronShortAddress):mutable -> value: u64 --Recommended to allow any address to be checked, but acceptable for only self_address to be supported
* X, claim_transfer(token_owner: NeutronAddress, id: u64, to: NeutronAddress) -> value: u64

Recommended Standard Functions:

* 2, store_state(key: [u8], value: [u8]):mutable
* 4, load_external_state(address: NeutronShortAddress, key: [u8]):static -> value:[u8]

Recommended Token Functions:

* X, set_token_balance(id: u32, address: NeutronShortAddress, value: u32); -- only usable by the token owner
* X, transfer_token_balance(token_owner: NeutronAddress, id: u64, value: u64) -> new_balance: u64

Extension Functions:
* 5, store_input_comap(prefix: [u8], ...):comap, mutable -- stores every comap value into storage, preserving type info with the data, and lists the data under the given keynames in the comap, with the given prefix attached to each key name
* 6, store_result_comap(prefix: [u8], ...):comap, mutable -- same as above
* 7, load_comap(prefix: [u8], ...):comap, static -> (...) -- loads all of the storage keys listed in the caller's output comap (data is ignored), with the prefix given prefixing each key name, into the caller's result comap 

## Details

Token balance state is treated internally as the same concept as regular load/store state, but with special rules applied for setting the value when not the token owner. claim_transfer is equivalent to the following for reference:

    assert!(id & 0x8000_0000_0000_0000 == 0);
    let map_key = generate_token_map_key(owner, id);
    let value = pop_contract_output_key(comap.peek_context(0).input_comap, map_key); --note this will destroy the contract's input map key, the only time that the input comap is modified in such a way
    let from_balance = get_token_balance(owner, id, sender_address);
    let to_balance = get_token_balance(owner, id, self_address);
    assert!(value <= from_balance); --validated upon using comap.transfer_balance
    to_balance += value;
    from_balance -= value;
    let to_key = generate_private_token_key(id, to);
    private_store_state_as(owner, to_key, to_balance); //sets private state within the token owner 
    let from_key = generate_private_token_key(id, from);
    private_store_state_as(owner, from_key, from_balance);

Note that if the `id` of a token has the top bit set, then the token can not be transferred by these methods, but using get_token_balance can still be used. 

Token balance transfers are not reflected until an appropriate claim_transfer occurs. For example:

    //contract input includes a transfer_token_balance() for 1000 coins and it previously had a balance of 100 of this type of coin
    assert!(get_token_balance(token_owner, token_id, self_address) == 100);
    assert!(claim_transfer(token_owner, token_id) == 1000);
    assert!(get_token_balance(token_owner, token_id, self_address) == 1100);

Minting new tokens as the owner can be done as so:

    let value = get_token_balance(self_address, id, target_address);
    value += 1000; //mint 1000 new tokens and send to target
    set_token_balance(id, target_address, value);

The transfer_token_balance function works as so:

    assert!(id & 0x8000_0000_0000_0000 == 0);
    let balance = get_token_balance(token_owner, id, self_address);
    let old_value = contract_output_key(comap.peek_context(0).output_comap, map_key).unwrap_or_default();
    assert!(balance >= value + old_value);
    let map_key = generate_token_map_key(owner, id);
    value += old_value;
    set_contract_output_key(comap.peek_context(0).output_comap, map_key, value);



