# Toolchain Setup

Currently there are no compiled binaries provided for Qtum-x86. Code changes happen too often to gain any value from setting up a binary compilation and distribution process. Thus, currently the toolchain and Qtum-x86 itself must be compiled from source. A series of dockerfiles and containers are provided to greatly simplify the build process, which is otherwise quite complex with several dependencies and OS specific steps. The Docker file and some utilities can be downloaded from: https://github.com/qtumproject/qtum-docker/tree/master/proto-x86

To install from source, follow these instructions:
    git clone https://github.com/qtumproject/qtum-docker/tree/master/proto-x86
    cd qtum-docker/proto-x86
    ./install.sh

This will take about an hour to an hour and a half on most computers, as it involves compiling the entire toolchain image, as well as building the client and simpleabi images. 

Once the docker images are built, it is fairly easy to use. In order to simplify things, some helper bash functions are provided and are highly recommended for use. They will be used in this documentation, but are by no means a requirement. 

The helpers.sh file is the following:

    #!/bin/bash
function qx86start() {
    docker run --rm -v "${PWD}:/root/bind" --name qx86 -d qtum-alpine qtumd -regtest -logevents -printtoconsole
}
export -f qx86start

function qx86stop() {
    docker stop qx86
}
export -f qx86stop

alias qx86cli='docker  exec qx86 qtum-cli -regtest'

function qx86deploy() {
    docker exec -t qx86 qtum-cli -regtest createcontract `cat $1`
}
export -f qx86deploy

function qx86make() {
    docker run --rm -v "${PWD}:/root/bind" -w "/root/bind" qtumtoolchain-alpine make
}
export -f qx86make

function qx86simpleabi() {
    docker run --rm -it -v "${PWD}:/root/bind" -w /root/bind qtum-simpleabi -a "$1" -d -e
}

export -f qx86simpleabi


The file can easily be used in your current bash session by simply doing `source helpers.sh`






