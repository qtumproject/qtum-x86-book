# Libqtum Variables

All of these "variables" are actually fixed areas of memory. They are simply represented as structures and with various helper constants to simplify their usage and access. 

Some variables are purposefully oversized (such as a boolean in a 32-bit integer) to keep efficient alignment within the structures

## Execution Data

This is a read-only block of memory which contains data specifically about the current execution state. 

Note: 

    static const struct ExecDataABI* const qtumExec = (const struct ExecDataABI* const) EXEC_DATA_ADDRESS;

    struct ExecDataABI{
        //total size of execdata area
        uint32_t size;

        uint32_t isCreate;
        UniversalAddressABI sender;
        uint64_t gasLimit;
        uint64_t valueSent;
        UniversalAddressABI origin;
        UniversalAddressABI self;
        uint32_t nestLevel;
    } __attribute__((__packed__));

* qtumExec->isCreate -- a boolean which is true (1) if the current execution will end in a new contract being created
* qtumExec->sender -- The contract or address which caused this contract to be executed. If this contract was not executed by another contract, it is the first address input in the transaction (TBD: vout signing in the future)
* qtumExec->gasLimit -- The total gas available for the current execution
* qtumExec->valueSent -- How many Qtum tokens were sent to the current contract by the current invocation
* qtumExec->origin -- the address which caused this entire call stack of contracts to initially be executed. This is the first address input in the contract transaction
* qtumExec->self -- The address of the current contract being executed
* qtumExec->nestLevel -- How many calls exist "above" this one. For instance, if the current execution is C and the call tree is A -> B -> C, then the nest level would be 2. At A (where there are no contract calls above it), the nest level would be 0

## Blockchain Data

This is a read-only block of memory which contains all the data about the current state of the blockchain as of the currently in-progress block. 

    static const struct BlockDataABI* const qtumBlockchain = (const struct BlockDataABI* const) BLOCK_DATA_ADDRESS;

    struct BlockDataABI{
        //total size of blockdata area
        uint32_t size;

        UniversalAddressABI blockCreator;
        uint64_t blockGasLimit;
        uint64_t blockDifficulty;
        uint32_t blockHeight;
        uint64_t previousTime;
        qtum_hash32 blockHashes[256];
    } __attribute__((__packed__));


* qtumBlockchain->blockCreator -- The address which staked the coins to create the current block (or the first address that receives the coins from block generation in the case of proof of work for regtest)
* qtumBlockchain->blockGasLimit -- The total gas limit for the current block.
* qtumBlockchain->blockDifficulty -- The current difficulty of the block (for mining / staking)
* qtumBlockchain->blockHeight -- The current number of blocks in the blockchain
* qtumBlockchain->previousTime -- The timestamp of the previous block (to prevent potential race conditions which prevent contract executions from being batched before block creation)
* qtumBlockchain->blockHashes[] -- The hashes of the previous 256 blocks. 0 is the previous block, 1 is the one before that, etc. 



