<pre>
  QIP-XXX
  Layer: Blockchain Protocol
  Title: Remove Gas Price Limits
  Author: Jordan Earls earlz@qtum.info
  Comments-Summary: 
  Comments-URI: 
  Status: Draft
  Created: 2020-11-6
  License:
</pre>

***

## Abstract ##

Right now in Qtum, there is a mandated gas price minimum of 4 qutoshis and is controlled by a DGP contract. This QIP would see the removal of hard consensus restrains on the minimum gas price that can be accepted on the blockchain, and instead favor a softer network approach that does not concern the on-chain consensus protocol. Specifically, the aim is to keep the DGP contract and use it for "soft" network purposes. In essence, nodes will automatically reject mempool transactions violating the DGP specified minimum gas price, unless they specifically set an option on their node to opt-out and to use their own minimum gas price. Blocks containing transactions which are lower than the DGP specified minimum gas price will become valid with this QIP.

## Motivation ##

It has often been desired in Qtum and other blockchains to pay for smart contract fees using an alternative currency than Qtum. Specifically, being capable of paying fees with QRC20 tokens is an especially relevant direction. With the help of QIP-XXX1 it may be possible to approach a method by which gas can be metered fairly yet also given a custom gas price that uses an alternative currency. 

## Specification ##


## Compatibility ##


## Nongoals ##


## Acknowledgments ##


## References ##



## Copyright ##

Q
