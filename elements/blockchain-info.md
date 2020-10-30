# Blockchain Info Element API

This element provides basic generic information about the current blockchain the smart contract is executing on.

ABI Information

Element ID: 1
Type: Neutron Standard

Functions:

* 1, block_creator() -> NeutronShortAddress
* 2, block_creator_long() -> NeutronFullAddress
* 3, last_difficulty() -> u64
* 4, block_gas_limit() -> u64
* 5, block_height() -> u64
* 6, block_time() -> u64
* 7, blockchain_info() -> struct
* 8, blockchain_vendor() -> u32
* 9, blockchain_id() -> u32


## Details

### Block Creator

This is the principle address which can be said to be responsible for creating the current block. If there is no direct creator of a block, or if there are multiple co-creators, then this may return a null address.

### Difficulty

The exact meaning of this is completely blockchain independent. The only constraint is that a higher number returned should mean a higher difficulty level or higher amount of competition in block creation

### Gas Limit

### Block Height

### Block Time

### Blockchain Info

The blockchain info returned is a structure which conveys the following information (TBD)

* Consensus mechanism -- PoW, PoS, DPoS, PoA, other
* Expected block time, in seconds (0 for sub-second or irregular blocks)


### Blockchain Vendor and ID




