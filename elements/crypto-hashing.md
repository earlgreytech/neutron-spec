# Cryptographic Hashing Element API

This element provides cryptographic one-way hashing algorithms for use within smart contracts.

ABI Information

Element ID: 8
Type: Neutron Standard
Provision: Mandatory 

Standard Functions:

* 1, sha256(input: [u8]):pure -> digest: [u8: 256]
* 2, sha3(input: [u8]):pure -> digest: [u8: 256]
* 3, hash(algorithm: u32, input: [u8]):pure -> digest: [u8]
* 3, hash_with_salt(algorithm: u32, input: [u8], salt: [u8]):pure -> digest: [u8] 
* 4, digest_length(algorithm: u32):pure -> length: u32 --shall return 0xFFFF_FFFF if length is variable

# Details

This element provides for methods to do 2 built in algorithms, SHA256 and SHA3. It also provides the generalized "hash" functions which can be used to extend this element for future hashing algorithms. These extended hashing functions make use of an Algorithm ID. The following Algorithm IDs are proposed:

* 1, sha256
* 2, sha3
* 3, rimpend160
* 4, sha1
* 5, poseidon
* Likely many others...

The blockchain can make a decision here in how unsupported algorithms are handled though:

* Generate an "unrecoverable" error, similar to if out-of-rent data were access. This would kill execution immediately reverting all changes. This would not allow for smart contracts to react to an invalid algorithm ID. However, it would greatly simplify the fork behavior around the addition of new algorithms. 
* Generate a recoverable error. This is a typical error that the smart contract can detect and handle. 




