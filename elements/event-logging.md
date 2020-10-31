# Event Logging Element API

This element provides the ability to log that an event happened which is accessible to external smart contract interfaces without needing to actually execute every smart contract. Ideal usage examples include successful trade notifications from a DEX, or a token transfer into your own account, etc. Note that logged events can not be accessed within smart contracts, though can be verified after the fact by using block header data proofs. 

ABI Information

Element ID: 5
Type: Neutron Standard
Provision: Recommended if underlying blockchain has the capability

Standard Functions:

* 1, log_event(key: [u8], data_count: u32, data:[u8]...):static 

Optional Functions:

* 2, log_event_concerning(address_count: u32, address: NeutronShortAddress..., data_count: u32, data:[u8]...):static

## Details

### Concerning

The concerning tag would allow for a list of addresses to be provided that can be watched by clients who opt-in. This requires specific consensus design to implement properly however. 

