# Libqtum Functions

## Error and revert 

    void qtumError(const char* msg) __attribute__ ((noreturn));

This function will immediately end contract execution and revert all state changes. This will NOT consume all gas available in the current execution. The message attached will create an [Event] with "error" as the key, and the message as the value, both as string types. If the message is `NULL` then no event will be created. Note that the error event is not capable of being read by a calling contract and is only intended for human debugging purposes. The event message will consume some gas, and so if low gas usage is critical, the message should be `NULL`.

## Stack operations

The following operations deal with the Shared Contract Communication Stack, or SCCS. This is a unique area of memory available to contracts which is used to communicate with contracts across execution borders. For instance, it would be used to store and pass argument data to a contract which is to be called from another contract. Typically, these operations are rarely used directly, as the [SimpleABI] code generation handles the contract interfaces on top of this stack. 



    void qtumPopExact(void* buffer, size_t size);
    size_t qtumPop(void* buffer, size_t maxSize);
    void qtumPeekExact(void* buffer, size_t maxSize);
    size_t qtumPeek(void* buffer, size_t maxSize);
    void qtumPush(const void* buffer, size_t size);
    size_t qtumPeekSize();

These are the primary push/pop/peek operations. The "Exact" variants will throw an error if no stack item is present, or if it is not the exact size specified. The other versions will only populate the given buffer up to maxSize and then return the actual size of the stack item. All variants of peek and pop operations will throw an error if no stack item is present. The `qtumPeekSize` operation will only return the size of the item on the stack, or throw an error if no item is present. 
    
    int qtumDiscard();
    int qtumStackClear();

The discard operation is used to discard a stack item without accessing it's contents. This operation is significantly cheaper in gas than using pop with a 0 size buffer. Discard will throw an error if no stack item is present. The stack clear operation will remove all items from the stack. It will never throw an error. 

    uint8_t qtumPop8();
    uint16_t qtumPop16();
    uint32_t qtumPop32();
    uint64_t qtumPop64();
    void qtumPush8(uint8_t v);
    void qtumPush16(uint16_t v);
    void qtumPush32(uint32_t v);
    void qtumPush64(uint64_t v);

These are useful variants of the push and pop operations which present a clean interface for common integer sizes without needing to pass around buffers etc. These all use the "exact" variant of pop and will throw an error if the size of the item on the stack is not the exact size of the specified integer type. 

    int qtumStackCount();
    size_t qtumStackMemorySize();

The StackCount operation is used retreive the number of items currently present on the stack. The StackMemorySize operation gives an indication of how much memory is currently being used by all items on the stack.


The SCCS stack has the following limitations:

* Total memory size cannot exceed 128Kbytes
* The size of an individual stack item can not exceed 64Kbytes
* The number of items in the stack can not be greater than 1024

There is one exception to the maximum memory size. Transaction data is passed into the contract via the SCCS. This data can exceed 128Kbytes in total, but still has the 64Kbyte per-item limitation. When the data is greater than 128Kbytes, no error is triggered until more data is attempted to be pushed by the executing contract. 

If a called contract's execution reverts or encounters an error, the SCCS will be cleared before returning execution to the caller. This is to prevent reverted state from "leaking".

## State operations

Qtum-x86 state, unlike the EVM, has dynamic length key and value state available to contracts. Qtum-x86 also uses a strict byte-size gas metering. Storing 100 bytes of "0" is not free, for instance. A state key is cleared by setting it's size to 0. Key and value sizes are weighted equally for gas metering. There is also a static fixed-cost to adding or updating any key. Clearing a state key is free, but does not enact a gas refund (TBD). 

Gas metering works in a way similar to the recent "net metering" method introduced to Ethereum. The first operation on a particular state key is more expensive than subsequent operations. This includes reading the same key several times, rather than just writing to the key. If a called contract accesses a key and then gives an error however, the read/write will effectively have "not happened", meaning the next access of that key will be considered the first.


    size_t qtumStore(const void* key, size_t keysize, const void* value, size_t valuesize); 
    size_t qtumLoad(const void* key, size_t keysize, void* value, size_t maxvalueeize); //returns actual value size
    void qtumLoadExact(const void* key, size_t keysize, void* value, size_t valueeize);

Similar to the stack operations, there are two variants of load operations. Exact and inexact. The exact variant will error if a state value is not exactly the size expected. The inexact version will write the value to the provided buffer up til the max size, and return what the actual size of the state key is. Using a NULL buffer to qtumLoad along with a max size of 0 can be used to cheaply determine the size of a state value. 

    size_t qtumLoadExternal(const UniversalAddressABI* addr, const void* key, size_t keysize, void* value, size_t maxvalueeize);
    void qtumLoadExactExternal(const UniversalAddressABI* addr, const void* key, size_t keysize, void* value, size_t valueeize);

In Qtum-x86 it is possible to directly read external contract state. There is no need for the contract to be executed, nor for a special access function be implemented. Note this can only be done with x86 contracts. Any non-x86 address specified will never have any key (ie, value size will always be 0). Note that this should be used with care for reading "internal" states of an external contract, as any future upgrades could break the expectations for what a key might hold. 

    int qtumUpdateBytecode(const void* bytecode, size_t bytecodesize);

This function is used to update the bytecode of the current contract. This will error if the contract is marked as being "non-upgradeable". Nothing immediately happens once upgrading the contract. If logic from the new bytecode is needed, a two-step upgrade should be used where the bytecode is first upgraded, then the contract executed again (with the new bytecode now) to carry out any needed migration or initialization steps. 


## Misc

    int qtumSelfCalled();

This will return 1 if the current contract has already been called and is in progress of execution within the current execution. This is used to detect and react to self-reentrancy, a common attack vector. It is recommended to handle reentrancy with great care. If safety is too difficult or cannot be guaranteed, then this function can be used to disallow self-reentrancy in the contract.

    int qtumCallContract(const UniversalAddressABI *address, uint32_t gasLimit, uint64_t value, QtumCallResultABI* result);

This function will cause execution to switch to another contract. Note that argument and return data for called contracts is placed in the SCCS. 

    int qtumParseAddress(const char* address, UniversalAddressABI *output);

This will take a base58 format address as a string and convert it to a UniversalAddress for easy usage in contract system calls. Note that this should be replaced with a library etc, as the base58 address parameters are different between testnet and mainnet environments. Usage of this function guarantees a contract can be used in either environment and will properly handle mismatched network addresses as an error. 

    uint64_t qtumGetBalance(const UniversalAddressABI* address);

This simply returns the balance of any contract on the blockchain. 

