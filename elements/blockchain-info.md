# Blockchain Info Element API

This element provides basic generic information about the current blockchain the smart contract is executing on.

ABI Information

Element ID: 1
Type: Neutron Standard
Provision: Mandatory in all Neutron implementations

Standard Functions:

* 1, block_creator():const -> NeutronShortAddress
* 2, block_creator_long():const -> NeutronFullAddress
* 3, last_difficulty():const -> u64
* 4, block_gas_limit():const -> u64
* 5, block_height():const -> u64
* 6, block_time():const -> u64
* 7, blockchain_info():const -> struct
* 8, blockchain_vendor():pure -> u32
* 9, blockchain_id():pure -> u32


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
* UTXO or Accounting



### Blockchain Vendor and ID

The blockchain vendor and ID are methods of uniquely identifying a blockchain within the smart contract environment. Vendor in this case should be more like the organization or type of chain. For instance, Ethereum and a private version of Ethereum should share the same vendor ID, but have different blockchain IDs. 

These are self selected IDs and there is no standard other than the pre-established ones within this document and that a best effort should be applied to prevent collisions. 

Vendor:

* 1, Qtum Foundation

Blockchain IDs:

* 1, Qtum Mainnet
* 2, Qtum Primary Testnet
* 3, Qtum Regtest
* 4, Qtum Neutron Testnet



