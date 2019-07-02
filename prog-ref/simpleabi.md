# SimpleABI Format

SimpleABI is a simplified format which is used for cross-contract communication in Qtum-x86, as well as for human interactions with a smart contract. SimpleABI is not defined by any consensus algorithm, and thus can easily be modified and/or replaced in the future. It also can co-exist with other ABI formats in the future, given there is some way of specifying which ABI format is being used. 

SimpleABI is designed to work specifically within Qtum-x86. Qtum-x86 uses the concept of a Smart Contract Communication Stack (SCCS), rather than a simple flat array of data. This concept allows for data to be encoded in a richer way, without requiring more work on parsing that data. Basically, the SCCS is a simple stack onto which data can be pushed and popped. This stack is persistent across smart contract calls, and thus can be easily used to pass dynamic amounts of data between smart contracts. There is of course some limits to this stack, such as overall stack size and memory usage, but it is expected that most smart contracts will have no problem with managing the stack. There is no rule, but a general good practice is to clear the stack of all data when ABI data is completely parsed. This minimizes the stack size and also clears any data that is otherwise unneeded by the contract. 

There are three main components to SimpleABI:

* The actual ABI definition for interacting with a contract
* The machine operations used in order to create and parse ABI data within a smart contract
* The User/API accessible SimpleABI data generation, for human/off-chain interactions with smart contracts


The ABI definition, or Interface Definition, is a simple text based format which also has the benefit of illustrating the state of the SCCS upon calling and returning from a contract. Here is a simple example:

    :name=MyInterface
    arg1:uint32 arg2:uint8 MyFunction:fn -> ret1:uint32 ret2:uint64

The `:name=` line is a directive setting the contract name. The contract name functions as a prefix to function names to avoid two contracts with a function like `add` generating the same function signature. 

`arg1:uint32` defines an argument by the name of `arg1` with the type "unsigned 32 bit integer". `arg2:uint8` defines an argument by the name of `arg2` with the type "unsigned 8 bit integer". Finally, `MyFunction:fn` defines the function signature which shall have the name of `MyFunction`. The actual size of a function signature is 4 bytes, or a 32 bit integer. `ret1:uint32` specifies that the function will return a 32 bit integer named `ret1`. And then, `ret2:uint64` specifies the function will also return a 64 bit integer named `ret2`. 

The actual operation to setup the calling of the external contract would look like this:

1. Push 32 bit integer (arg1)
2. Push 8 bit integer (arg2)
3. Push 32 bit integer (function signature)
4. Call external contract address

The external contract, would then use the following logic in order to parse this ABI data:

1. Pop 32 bit integer (function signature)
2. Determine that function signature matches that of MyInterface::MyFunction
3. Pop 8 bit integer (arg2)
4. Pop 32 bit integer (arg1)
5. (recommended, but not mandatory) clear the SCCS of any remaining data
6. Run the logic for MyFunction

Once the logic for the MyFunction code has been run, it will then do the following to satisfy the Interface Definition provided:

1. Push 32 bit integer (ret1)
2. Push 64 bit integer (ret2)
3. Terminate contract execution with successful status

Control is then transferred back to the calling contract which then receives the returned ABI data from MyFunction:

1. Ensure contract execution was a success
2. Pop 64 bit integer (ret2)
3. Pop 32 bit integer (ret1)
4. (recommended, but not mandatory) clear the SCCS of any remaining data

As you can see with this explanation, the Interface Definition literally describes the items which should exist on the SCCS. It is of course up to SimpleABI consumers to ensure the data generated is correct, and that the data received is correct, and to not expose any security problems or unexpected behavior due to incorrect ABI data being given. 

The function signature of the `MyFunction` above would be the first 4 bytes of the following string, when hashed by SHA256:

    uint32 uint8 MyInterface_MyFunction -> uint32 uint64

With this, it is thus possible to have multiple MyFunction names taking different arguments and with different respective code being activated. Currently this is not recommended however, as the C code generation can not accomadate this due to limitations in the C language. Another important note is that the names of arguments can safely be refactored without breaking the function signature. Only the data types of arguments and return values, the interface name, and the function name are used in the function signature and thus can not be changed without potentially affecting compatibility. 

The total list of basic types is as follows:

* int8
* uint8
* int16
* uint16
* int32
* uint32
* int64
* uint64
* uniaddress

The `uniaddress` type encodes a 36 byte UniversalAddress value. The remainders are simple integer types that are signed or unsigned. It is also possible to specify an array of a type. For instance:

    values:uint32[] MyFunction:fn -> returnValues:uint8[]

And finally, there is a `void` argument to indicate that no return values are given. For instance, this would represent a function which accepts no inputs and returns no outputs:

    MyVoidFunction:fn -> void

The `fn` item is always needed so that the external contract knows which function to execute, so `void` is not needed on the left hand side of the `->`. 

There are also some function attributes available:

* `payable` -- indicates the function is capable of accepting payment
* `nonpayable` -- indicates the function is not capable of accepting payment (default)

This is an example usage:

    sendTo:uniaddress amount:uint64 giftCoins:fn:payable -> void

Note that the function attributes does NOT go into the function signature, and there is no mandatory enforcement requirement of any function attribute. The function attributes are solely for informative purposes. It is possible to specify a function is nonpayable, but then still give it payment if the function does not implement logic to error upon payment received. These function attributes are however used by the code generator to provide a proper API which does not expose a payment option for a function marked nonpayable, and also will implement code in the ABI parsing routines to throw an error if payment is sent to a function marked nonpayable. 

Because of the informative and non-mandatory enforcement on function attributes, it will be possible to also add custom attributes which may work to implement special logic in some code generators, or simply used to inform humans of a nuance with the ABI. For instance:

    StopContract:fn:admin-only -> void

The `admin-only` attribute would not disallow a person from actually calling that function, but it says to people using the ABI that some special permission is required which most users will not have. With these custom attributes, whatever behavior is expected though will be required to be implemented in the function. The code generator will simply ignore such things. 

Proposed official attributes include:

* static
* pure
* owner-only
* admin-only


There is one other interesting directive to go along with `:name`. This is the `:implements` directive. This specifies that the current interface also implements another interface. For instance, there could be something like so:

    :name=CrowdsaleToken
    :implements=StandardToken
    buyCoins:fn:payable -> amount:uint64

This defines a CrowdsaleToken interface which also implements the StandardToken interface. The functions defined within StandardToken will still use the `StandardToken` interface name. Thus, if a user (or contract) only knows about the StandardToken interface, and unknowingly interacts with a CrowdsaleToken interface, the StandardToken interface will still function as expected. 

In order to generate code or data for the ABI, the program being used must somehow learn what StandardToken is. This is currently considered "will change later", but for right now there is plans to have a standard search directory, and to use the current directory the interface file is located in in order to search for a file named StandardToken.abi. It is also possible to explicitly specify where the ABI file should be retreived from:

    :implements=StandardToken(https://raw.githubusercontent.com/qtumproject/StandardInterfaces/master/StandardToken.abi)

Or using a local path:

    :implements=StandardToken(../interfaces/StandardToken.abi)

It is possible for a single interface to contain multiple `:implements` directives, and any interface implemented which contains it's own `:implements` directive will also be implemented by the top-level interface.

There are some other directives planned as well, such as `:version`, but these are not yet implemented nor defined. 

Finally, the ABI data generation program will simply take inputs, a choice of which function is being used, and output a flat hex string. Internally within Qtum-x86, the data to be given to a contract must be flattened into a single string of data. In order to convert this into data which is pushed onto the SCCS, a simple length prefix is given to each item to be pushed. The length prefix determines how much of the string is pushed, and then the next item's length is read and it's data pushed, and so on. The data generation program is still under construction will also double as a programmatic ABI capable of taking JSON input data, an ABI definition, and providing the hex string which can then be placed into the Qtum-x86 blockchain for interacting with smart contracts. This is expected to be used by Dapps and other user friendly interfaces on top of a smart contract system. 


## SimpleABI Code Generation for C

The C code generator is somewhat clunky, due to C being a fairly clunky language itself. Any dynamically sized data (such as arrays) will be copied into memory which is allocated by `malloc` and all other data will simply be held in stack space.

Let's take the following interface definition:

    :name=ExampleABI
    arg1:uint8 arg2:uint32 argUA:uniaddress SimpleFun:fn -> ret1:uint8

If you are wishing to implement this ABI interface in your smart contract, then you would use the `--decode` option of SimpleABI. This would generate some code which should be compiled into smart contract program. Lets look at the header file generated by this for the definition:

    #ifndef ExampleABIABI_H
    #define ExampleABIABI_H

    //Function IDs
    #ifndef ID_ExampleABI_SimpleFun
    #define ID_ExampleABI_SimpleFun 0xa1b6e0f6
    #endif


    void dispatch();

    void ExampleABI_SimpleFun(uint8_t arg1, uint32_t arg2, UniversalAddressABI* argUA, uint8_t* ret1);


    #endif

The `dispatch` function is important. It should be placed in the `main()` function of your smart contract after any special initialization that is needed. The `dispatch` function will parse the ABI data and then call `ExampleABI_SimpleFun`. This function is not created by the code generator, and so without implementing it, you will receive a linking error about the missing symbol. It is simple to implement it:

    void ExampleABI_SimpleFun(uint8_t arg1, uint32_t arg2, UniversalAddressABI* argUA, uint8_t* ret1){
        //logic here
    }

C is not capable of actually returning more than one piece of data at a time, so instead a pointer is used so that the data to be returned can be set from within the function. In addition, any argument which is not a simple integer will use a pointer. This is because otherwise it could require a significant amount of memory copying as well as stack space, wasting gas and computational resources. 


If you are not wishing to implement this interface in your contract, and instead wish to call an external contract which implements it, then you would use the `--encode` option. The header file for this use case would look like so:

    #ifndef ExampleABIABI_H
    #define ExampleABIABI_H

    //Function IDs
    #ifndef ID_ExampleABI_SimpleFun
    #define ID_ExampleABI_SimpleFun 0xa1b6e0f6
    #endif


    QtumCallResult  ExampleABI_SimpleFun(UniversalAddress __address, QtumCallOptions* __options, uint8_t arg1, uint32_t arg2, UniversalAddressABI* argUA, uint8_t* ret1);


    #endif


The first two arguments are `address` and `options`. The address is of course the external contract address to execute, while the `QtumCallOptions` is a structure which allows for various options to be set for the call, such as the gas limit, and potentially making the call a static call, etc. The remaining arguments are of course similar to the dispatch function. Though in this case, it is your own code which must setup the variables and such used for receiving the external variables via pointer. 
