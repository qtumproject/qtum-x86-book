# What is Qtum-x86

Qtum-x86 is the latest prototype from Qtum. It introduces the x86 Virtual Machine. Currently you can write smart contracts in C, but more languages will be supported in the future. This prototype is strictly a preview and the contract interface is subject to change before final release. 

This version of Qtum comes with the following restrictions:

* Must be used from Docker or compiled from source
* EVM contract usage is not supported
* The testnet and mainnet networks can not be used and will not sync. Only regtest is supported
* Orphaning blocks and reorganizing the blockchain will not work properly.
* Only the command line RPC interface is supported for x86 contracts, though the GUI will still work

Many things are still yet to be implemented, and bugs may exist. Bug reports are appreciated at our [github](https://github.com/qtumproject/qtum/issues).



