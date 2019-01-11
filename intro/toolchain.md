# Toolchain Setup

Currently there are no compiled binaries provided for Qtum-x86. Code changes happen too often to gain any value from setting up a binary compilation and distribution process. Thus, currently the toolchain and Qtum-x86 itself must be compiled from source. A dockerfile is provided to greatly simplify the build process, which is otherwise quite complex with several dependencies and OS specific steps. The Docker file and some utilities can be downloaded from: https://github.com/qtumproject/qtum-docker/tree/master/proto-x86

The docker file is initially built as so:

    docker build -t qtumx86 -f Dockerfile .

This will take several hours on most computers, as it involves compiling the entire toolchain, twice (once for freestanding, once for Qtum) as well as Qtum itself. 

To rebuild the image from scratch (such as to update the version etc), use the `--no-cache` option:

    docker build -t qtumx86 -f Dockerfile . --no-cache

Once the docker image is built, it is fairly easy to use. In order to simplify things, some helper bash functions are provided. They will be used in this documentation, but are by no means a requirement. 

The helpers.sh file is the following:

    #!/bin/bash
    function qx86start() {
        docker run --rm -v "${PWD}:/root/bind" --name qx86 -d qtumx86 qtum/src/qtumd -regtest -logevents
    }
    export -f qx86start

    function qx86stop() {
        docker stop qx86
    }
    export -f qx86stop

    alias qx86cli='docker exec qx86 qcli'

    function qx86deploy() {
        docker exec -t qx86 deploy_contract `hexdump -e \"%x\" $1` 
    }
    export -f qx86deploy

    function qx86tb() {
        docker run --rm -v "${PWD}:/root/bind" qtumx86 x86tb
    }
    export -f qx86tb

    function qx86make() {
        docker run --rm -v "${PWD}:/root/bind" qtumx86 qmake "$@"
    }
    export -f qx86make


The file can easily be used in your current bash session by simply doing `source helpers.sh`






