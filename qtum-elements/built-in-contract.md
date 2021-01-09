# Built In Contract

This is not actually an element API but is rather the owner of the underlying blockchain asset, QTUM. 

ABI Information

Address: Version 0x8100_0000, data: 0
Type: Qtum Standard

All ABI data is ignored except for token balance transfers owned by this version. It will automatically claim all transfers for this address, thus allowing for pubkeyhash and other Qtum specific address types to have their balance modifications committed. All Qtum tokens within the smart contract system are owned by this contract using a token_id of 0.

When tokens are sent to a smart contract from a Qtum transaction, ABI data for that transfer is added using this address as the token_owner. 

The AAL code will pick up all modified state within this contract at the end of Neutron execution and construct a necessary transaction in order to send appropriate data to pubkeyhash etc addreses. Outstanding and unclaimed transfers at the end of a contract execution will result in implicitly updating the balance of the sender of those tokens in order to generate a refund at the end of Neutron execution.