# Creating a simple State Registry

In this our goal is to create a simple state registry. This assumes some experience with C or C++, and at least having the Hello World example working on your system. 

First, the design for our registry must be laid out. To keep things simple, we will simply have each state key in the registry have a name and an owner. The value will be a 32 bit integer. The value can be read by anyone, but only updated by the owner who created it. It will not be possible to have two state keys with the same name owned by two different users

The interface definition will we use is as so:

    :name=WalkthroughStateRegistry

    name:uint8[] value:uint32 set:fn -> void
    name:uint8[] getOwner:fn -> owner:uniaddress
    name:uint8[] get:fn -> value:uint32

Next, the exact way of dividing up state should be determined. We need to track 2 pieces of data per key, the owner, and the actual value. It is possible to store these are two separate keys like so:

* value_%name% -> value
* owner_%name% -> owner

However, there is (at least planned) some extra expenses per key consumed. Thus, unless the data pieces will be updated separately or are especially large, it is typically cheaper in gas costs to use a single key to store both pieces of data, like so:

* %name% -> owner+value

In this, rather than the value of the state key being a single value, we store both values. We can easily represent this using a structure in C:

    typedef struct{
        UniversalAddress address;
        uint32_t value;
    } __attribute__((packed)) ownerValue_t;

Note that the `__attribute__((packed))` specifies the structure should be "packed", ie, without extra bytes inserted for optimized alignment. These extra bytes would increase the storage size and thus gas usage. However, more importantly, with different compiler versions and even different optimization options this structure could look different in terms of raw size and the exact layout of the bytes within this structure. By specifying it should be packed, we guarantee the compiler doesn't try to be clever with how this is laid out and keeps the exact layout we specify.

Now, we will begin our project structure. We need four files to begin with:

* Makefile -- automated instructions for compiling the project
* StateRegistry.c -- The actual implementation of our contract
* StateRegistry.h -- The structure definitions of cour contract (this is optional here, but a requirement for projects with multiple implementation files)
* WalkthroughStateRegistry.abi -- our ABI definition for the contract






