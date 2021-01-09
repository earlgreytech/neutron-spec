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
* X, claim_transfer(token_owner: NeutronAddress, id: u64, to: NeutronAddress) -> value: u64

Standard Optional Functions:

* 3, key_exists(key: [u8]):static -> exists:bool
* 4, load_external_state(address: NeutronShortAddress, key: [u8]):static -> value:[u8]
* 5, store_input_comap(prefix: [u8], ...):comap, mutable -- stores every comap value into storage, preserving type info with the data, and lists the data under the given keynames in the comap, with the given prefix attached to each key name
* 6, store_result_comap(prefix: [u8], ...):comap, mutable -- same as above
* 7, load_comap(prefix: [u8], ...):comap, static -> (...) -- loads all of the storage keys listed in the caller's output comap (data is ignored), with the prefix given prefixing each key name, into the caller's result comap 


Note that NeutronDB as it is currently designed would not implement the `key_exists` function

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





----
Alt Alt idea, use comap for transferring allowances

CoMap gets a special reserved space, key prefix of 0 which can not be modified nor read by ordinary means
This would allow multiple token transfers in a single go by writing to the output ABI map and then calling a smart contract, while keeping as close as possible to the current ETH model of payment transfers

Basic method:

comap.transfer_balance(token_owner, token_id, to_contract, value); //creates a new key within comap, `0 : token_owner : token_id -> value`
call(to_contract);

to_contract code:
global_state.claim_transfer(token_owner, token_id) -> value; //destroys the key in the input map, adding it to the current smart contract's GlobalState, ie, writing to global key `token_owner : TOKEN_SPACE : contract_address : token_id` and the counterpart key containing `msg.sender` to deduct the amount transferred

If the to_contract at this point reverts then that state modification would revert with it. The comap output map would also be destroyed by a revert, freeing the claimed and unclaimed sender balance coins. Note: all balance() etc type calls would check the output comap for affecting balance transfers. This makes transfers properly reserve the proper amount of balance and would return that balance if unclaimed (or reverted) upon call return

In order to mint coins using this approach, something like this would be used ONLY by the token_owner:

let value = global_state.get_balance(token_owner, token_id, address).unwrap_or(0);
value += 1000;
global_state.modify_token_balance(token_owner, token_id, to_address, value); 

This could of course be used for destroying or creating new tokens. 

Any `token_id` with the top bit set will be regarded as "by owner permission only" and the transfer_balance and claim_transfer functions would result in an error. However, the following function could still be used as a unified balance check:

global_state.get_balance(token_owner, token_id, address) -> value;

