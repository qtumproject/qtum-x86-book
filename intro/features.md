# New Features in Qtum-x86

Qtum-x86 implements many features not supported for neither Ethereum's EVM nor Qtum's version of EVM. Many includes features expose the full power of the Qtum Account Abstraction Layer, making it more natural for contracts to interact with the UTXO model of Qtum. Other features are ones more natural in x86 or would be very difficult to implement with the EVM's design. 

## UTXO Interaction

Qtum-x86 contracts can directly interact with the various pieces of the transaction in which they are included. This can be used especially for a contract to "withness" an otherwise normal payment. The contract in this case would read what address (or addressses) spent coins in order for them to be sent to another address. The contract can log this action or do other processing, such as automatically releasing XRC20 tokens to the sender. This is possible in an EVM contract but is more complicated, and sending to different address types would require special care and otherwise abnormal transactions

## Contract Events

One of the greatest iterative improvements over the EVM is the implementation of contract events in Qtum-x86. Events ("logs" in Ethereum) are bits of data that are not saved directly into the blockchain, but can be proven to exist within it. In Ethereum this data is of a strict format consisting of up to 4 256 bit "topics" and 1 256 bit "data" field.. Thus, 256 bit hashes is typically used for any data larger than 256 bits. 

In Qtum-x86 events are flexible dynamic length key-value data with a helpful type ID for both fields. This allows for contracts to create data that can be easily read in it's intended presentation (ie, as an address, number, string) by users without special tooling or coding knowledge.  

## Contract Upgrades

Contract upgrades are trivial in Qtum-x86 when compared to Solidity/EVM contracts. The biggest factor that contributes to this success, it is possible to modify contract bytecode within the contract itself. This eliminates the need for the infamously tedious proxy contract pattern in Solidity. There is no need to split state, address, and bytecode into separate contracts. This can be used for cheaper contract upgrades than otherwise as well. If the contract upgrade is very small, it might be possible to patch only a single area of bytecode. This reduces the amount of transaction size bloat and thus gas costs for such upgrades. There will later be an option to disable contract bytecode upgrades, making contracts capable of opting in to 100% immutability of their bytecode. 

The other contributing factor to easier contract upgrades is the explicit handling of state. In the EVM both key and value data in contract state is constricted to a 256 byte limit. This requires the Solidity compiler to automatically generate the actual key data used from a Solidity contract. This is a topic of constant improvement, but for the most part this prevents you from doing radical restructuring of how the code is layed out in a contract or doing major refactorings involving renaming of storage variables. In Qtum-x86 storage is managed explicitly by contracts without a "variable" abstraction. The dynamic length of key and value data allows for complex structures to be stored as either a key or a value, and thus allowing easy direct usage of Qtum-x86 storage. This allows for refactorings and upgrades to be trivially managed without great restriction. It also allows for easier data migration into different formats. 

The key data restriction in the EVM also contributes to a security problem not present in Qtum-x86. In the EVM, if some storage based array variable can be indexed to arbritrary values, it is possible to read or write to any variable that is "later" in the 256 bit storage key space. This vulnerability does not exist in Qtum and thus is it safe to allow for dynamic storage array indexing. 

## Universal Address System

In Qtum there are many different types of addresses. This is strictly different from Ethereum, where only a single type of address exists which may be either a contract or public key depending on blockchain state. In Qtum there are several types:

* Pay to pubkey
* Pay to pubkeyhash
* Pay to scripthash
* Pay to witness pubkeyhash
* Pay to witness scripthash
* EVM contract
* x86 contract
* (potentially other VM addresses later or on private blockchains)

Previously in Qtum's EVM machine, it was only possible to interact with either EVM contracts or pay to pubkeyhash addresses. This was due to the design of the EVM which only allows 160 bits for an address. It is not possible to encode an address and a "type" within 160 bits without compromising the security of the address and some address types in Qtum can not fit into 160 bits. 

In Qtum-x86 we introduce the concept of a UniversalAddress. This is an address which encodes both type and data, and is intended to be of consistent format in mainnet, testnet, regtest, and even forks of the Qtum blockchain. This is strictly different from the Base58 encoding of addresses in Qtum. The "type" there also doubles as an easy to read prefix for the user. It would be bad if it were easy for a user to send mainnet tokens to testnet addresses or to mistake a mainnet address as being from the testnet or from a different blockchain. Thus, each blockchain type uses different "prefixes". UniversalAddress on the otherhand is intended only for smart contract usage and should be mostly invisible to end users. Currently in Qtum-x86, the data portion of this address is fixed size, but in the future it will be of dynamic size. 

Along with the benefit of actually being capable of interacting with all of these address types, it is easily possible to filter and determine which type of address is being used without any blockchain information. 


## External Contract State Access

In Qtum-x86 it is possible to read directly from an external contract's state. There is no need for "getter" functions as in Solidity/EVM contracts. Instead it is expected that each contract would publish a public state "API" consisting of the various keys guaranteed to stay the same format in the contract, even persisting through contract bytecode upgrades.

## Shared Contract Communication Stack (SCCS)

A new memory area called the Shared Contract Communication Stack is available to Qtum-x86 smart contracts. This is a simple stack which all smart contracts in a single execution has access to. This stack is used for passing argument and return data between different contracts. The stack allows for multiple pieces of data to be managed separately, making it trivial to allow for dynamic numbers of parameters and return data. An ABI format is described later which has been built on top of this feature. 




