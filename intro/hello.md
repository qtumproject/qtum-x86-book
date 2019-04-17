# Hello World

After the toolchain is setup, we can finally test it by creating a simple Hello World smart contract.

This is the source code for a very simple Hello World contract:

    #include <qtum.h>

    int onCreate(){
        qtumEventStringString("Hello World", "Contract creation");
        return 0;
    }

    int main(){
        qtumEventStringString("Hello World", "Execution Success!");
        return 0;
    }


The `onCreate` function is called only when the contract is initially deployed to the blockchain using `createcontract` or `qx86deploy`. The `main` function is called anytime the contract is called, either by another smart contract, or by `callcontract` or `sendtocontract`. 

The `qtumEventStringString` function creates a new event upon execution. The `StringString` suffix means that the "key" of the event is a string and that the "value" is a string. There are also functions like `qtumEventStringInt64` for specifying a 64bit integer key rather than a string. Using this we can later retreive the event using `qx86cli searchevents`. 

The compilation process for this smart contract is a bit more complicated. We use the toolchain docker container in order to do the actual processing. This makes it simple to compile Qtum-x86 smart contracts on any operating system, rather than just Linux or OSX. 

First, we write a makefile. This makefile will be executed on the docker container so that all of the Qtum tooling is available. Here is the template we use for hello world:

    # files to be built
    # fill this in as a template
    HDRS = 
    C_SRC = helloworld.c
    OUTPUT = helloworld.elf
    LIBS = 
    BYTECODE = helloworld.qbit

    C_OBJS = $(subst .c,.o,$(C_SRC))

    #these default flags will just remove dead code and give warnings
    CFLAGS += -Wall -ffunction-sections -fdata-sections
    LDFLAGS += -Wl,--gc-section


    default: $(BYTECODE)

    $(BYTECODE): $(OUTPUT)
        x86testbench -assemble -raw $(OUTPUT) > $(BYTECODE)

    $(OUTPUT): $(C_OBJS)
        $(CC) $(LDFLAGS) -o $(OUTPUT) $(C_OBJS) $(LIBS)

    $(C_OBJS): $(HDRS) $(C_SRC)
        $(CC) $(CFLAGS) -c $*.c -o $@

    clean:
        rm -f $(C_OBJS) $(OUTPUT) $(BYTECODE)


Lets break the variables of this down:

    HDRS = 

This is to track any header files in the contract. Although header files are not compiled directly, these are tracked so that the code can be recompiled when any header file is changed.

    C_SRC = helloworld.c

These are the C files to be compiled. This can be expanded to multiple files by appending additional file names with a space separator. 

    OUTPUT = helloworld.elf

This is the output of the GCC compiler. This is a standard ELF file and is ideal for post-compilation analysis. Most existing reverse engineering and analysis tools support this format. Thus, if analysis is needed on the binary file from the compiler, this is the one to use. 

    LIBS = 

This is for adding static libraries to the contract.

    BYTECODE = helloworld.qbit

This is the actual bytecode file that is formatted for the Qtum blockchain. This is greatly simplified from ELF and although is a very easy to parse and use due to its limitations, no tools will support this (at least yet). When deploying this to the blockchain using `qx86deploy`, the Qtum RPC command "createcontract" is used, and the argument is this file which is dumped to a hex string. 

These are the important things to know about modifying this Makefile. For most simple contracts, this will be sufficient. With more complicated ones involving many libraries etc, it may be ideal to to write a custom makefile using this for inspiration. One last detail you should be aware of is that the easiest way to interact with the Makefile to ensure everything compiles would be to use `qx86make` rather than `make`. This sends your Makefile and C files up to the qx86 docker container where libraries are prepared and linked and ready for interaction. 

In order to actually deploy this contract, you can use this script in a local shell (assuming you sourced the "helpers.sh" script from the toolchain setup)

    qx86start
    qx86cli generate 600 # this will generate 600 blocks. This will start the chain and give us some coins to work with
    qx86deploy helloworld.qbit
    qx86cli generate 1 # generate 1 block so that we can mine our contract and ensure its processed

After this, you should see output like this:

    Jordans-MacBook-Pro:helloworld earlz$ qx86deploy helloworld.qbit
    {
      "txid": "02b28cfcc7d1fdcb5963ee2ff9024a045ac6fb4d8d66683b027e47fd5973ed0b",
      "sender": "qUJfCTe5LacTUBjC7djrEfoMtfZGXD7J4L",
      "hash160": "75e089080145b639bf496e27cc3ed655e13377d0",
      "hexaddress": "85840fafe5b51343f58259de8b48fe6b001cce57",
      "address": "xLUbyAmNUTRxLt67x5gDtDxBzWpLrH92Zn"
    }
    Jordans-MacBook-Pro:helloworld earlz$ qx86cli generate 1
    [
      "300b9d8d9e5d05c22d09ff91335766e559af21ab70ee6f37daf280220eecc06d"
    ]

There are a lot of different fields returned from the deployment command. The important one is `address` with a value of `xLUbyAmNUTRxLt67x5gDtDxBzWpLrH92Zn`. This will vary on different instances of the blockchain, so this will effectively be random. The `txid` field is the transaction ID which the contract was placed in. 

Now you can do the following:

    qx86cli searchevents

This will give a fairly large JSON output. There will be one for each contract execution, no matter if the contract failed or not. Here is a breakdown of the fields in the JSON:

* block-hash -- the block hash in which the execution occured. 
* tx-hash -- TBD
* tx-n -- TBD
* address -- the root contract address which was executed. This is either the address created by `createcontract` or called by `sendtocontract`
* used-gas -- how much gas was used by the execution
* sender-refund -- how many coins will be returned to the sender. In case of an error or revert, all coins sent in an execution will be refunded
* status -- the description string of the execution status
* status-code -- a numeric code for the execution status
* transfer-txid -- TBD
* commit-state -- Whether or not state is committed for this contract execution
* deltas -- All state changes which occur in this contract execution. This is attempted to be automatically parsed for strings or numbers and displayed in a more user friendly manner (note: might change later)
* deltas-raw -- the same as deltas, but is always displayed in raw hex form
* modified-balances -- all address balances which are modified by this execution
* spent-vins -- all of the UTXOs which will be marked as spent due to this execution
* events -- the actual events created by this execution. This is an array of key-value pairs
* calls -- this is a recursive structure where other execution results (this json structure) are placed within it. There is one result for each contract call which takes place within the overall execution. It is possible for results to be nested multiple levels (note: this is subject to change)

You'll see something like this at the bottom of the JSON output:

          "modified-balances": {
          },
          "spent-vins": [
          ]
        },
        "events": {
          "Hello World": "Contract creation"
        },
        "calls": [
        ]
      }
    ]

The hello world event for contract creation is seen here. 

In order to call the contract to test it (replacing the address with yours), do the following:

    qx86cli callcontract xLUbyAmNUTRxLt67x5gDtDxBzWpLrH92Zn 00

This will immediately return a JSON result. `callcontract` simply calls the contract locally and does NOT create a blockchain transaction. Thus, there are no fees etc associated with `callcontract`. In the JSON result you should now see `"Hello World": "Execution Success!"` in the events array. 

In order to do the same thing but on the blockchain as an actual transaction, do the following:

    qx86cli sendtocontract xLUbyAmNUTRxLt67x5gDtDxBzWpLrH92Zn 00
    qx86cli generate 1 #create one block to confirm the transaction

You will not see the large JSON result. In order to see it, you need to use `searchevents` again. 



