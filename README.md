ZeroTier SDK (beta)
======

The ZeroTier SDK offers a microkernel-like networking paradigm for containerized applications and application-specific virtual networking.

The SDK couples the ZeroTier core Ethernet virtualization engine with a user-space TCP/IP stack and a library that intercepts calls to the Posix network API. This allows servers and applications to be used without modification or recompilation. It can be used to run services on virtual networks without elevated privileges, special configuration of the physical host, kernel support, or any other application specific configuration. It's ideal for use with [Docker](http://http://www.docker.com), [LXC](https://linuxcontainers.org), or [Rkt](https://coreos.com/rkt/docs/latest/) to build containerized microservices that automatically connect to a virtual network when deployed. It can also be used on a plain un-containerized Linux system to run applications on virtual networks without elevated privileges or system modification.

More discussion can be found in our [original blog announcement](https://www.zerotier.com/blog/?p=490) and [the SDK product page](https://www.zerotier.com/product-netcon.shtml).

The SDK is currently in **BETA** and is suitable for testing and experimentation. Only Linux is supported. Future updates will focus on compatibility, full stack support, and improved performance, and may also port to other OSes.

## Building the SDK

The SDK works on Linux and has been lightly tested on OSX. To build the service host, IP stack, and intercept library, from the base of the ZeroTier One tree run:

    make clean
    make netcon

This will build a binary called `zerotier-sdk-service` and a library called `libztintercept.so`. It will also build the IP stack as `src/liblwip.so`. 

To enable debug trace statements for Network Containers, use `-D_SDK_DEBUG`

The `zerotier-sdk-service` binary is almost the same as a regular ZeroTier One build except instead of creating virtual network ports using Linux's `/dev/net/tun` interface, it creates instances of a user-space TCP/IP stack for each virtual network and provides RPC access to this stack via a Unix domain socket. The latter is a library that can be loaded with the Linux `LD_PRELOAD` environment variable or by placement into `/etc/ld.so.preload` on a Linux system or container. Additional magic involving nameless Unix domain socket pairs and interprocess socket handoff is used to emulate TCP sockets with extremely low overhead and in a way that's compatible with select, poll, epoll, and other I/O event mechanisms.

The intercept library does nothing unless the `ZT_NC_NETWORK` environment variable is set. If on program launch (or fork) it detects the presence of this environment variable, it will attempt to connect to a running `zerotier-sdk-service` at the specified Unix domain socket path.

Unlike `zerotier-one`, `zerotier-sdk-service` does not need to be run with root privileges and will not modify the host's network configuration in any way. It can be run alongside `zerotier-one` on the same host with no ill effect, though this can be confusing since you'll have to remember the difference between "real" host interfaces (tun/tap) and network containerized endpoints. The latter are completely unknown to the kernel and will not show up in `ifconfig`.


## Modes of operation

There are generally two ways one might want to use this SDK/service. The first approach is a *compile-time static linking* of our SDK/service directly into your application. With this option you can bundle our entire functionality right into your app with no need to communicate with a service externally, it'll all be handled automatically. The second is a service-oriented approach where our SDK is *dynamically-linked* into your applications upon startup and will communicate to a single ZeroTier service on the host. This can be useful if you've already compiled your applications and can't perform a static linking.

![Image](docs/img/methods.png)


## Linking into an application on Mac OSX

Example:

    gcc myapp.c -o myapp libzerotierintercept.so
    export ZT_NC_NETWORK=/tmp/netcon-test-home/nc_8056c2e21c000001

Start service

    ./zerotier-netcon-service -d -p8000 /tmp/netcon-test-home

Run application

    ./myapp


## Starting the Network Containers Service

You don't need Docker or any other container engine to try Network Containers. A simple test can be performed in user space (no root) in your own home directory.

First, build the netcon service and intercept library as described above. Then create a directory to act as a temporary ZeroTier home for your test netcon service instance. You'll need to move the `liblwip.so` binary that was built with `make netcon` into there, since the service must be able to find it there and load it.

    mkdir /tmp/netcon-test-home
    cp -f ./netcon/liblwip.so /tmp/netcon-test-home

Now you can run the service (no sudo needed, and `-d` tells it to run in the background):

    ./zerotier-netcon-service -d -p8000 /tmp/netcon-test-home

As with ZeroTier One in its normal incarnation, you'll need to join a network for anything interesting to happen:

    ./zerotier-cli -D/tmp/netcon-test-home join 8056c2e21c000001

If you don't want to use [Earth](https://www.zerotier.com/public.shtml) for this test, replace 8056c2e21c000001 with a different network ID. The `-D` option tells `zerotier-cli` not to look in `/var/lib/zerotier-one` for information about a running instance of the ZeroTier system service but instead to look in `/tmp/netcon-test-home`.

Now type:

    ./zerotier-cli -D/tmp/netcon-test-home listnetworks

Try it a few times until you see that you've successfully joined the network and have an IP address. Instead of a *zt#* device, a path to a Unix domain socket will be listed for the network's port.

Now you will want to have ZeroTier One (the normal `zerotier-one` build, not network containers) running somewhere else, such as on another Linux system or VM. Technically you could run it on the *same* Linux system and it wouldn't matter at all, but many people find this intensely confusing until they grasp just what exactly is happening here.

On the other Linux system, join the same network if you haven't already (8056c2e21c000001 if you're using Earth) and wait until you have an IP address. Then try pinging the IP address your netcon instance received. You should see ping replies.

Back on the host that's running `zerotier-sdk-service`, type `ip addr list` or `ifconfig` (ifconfig is technically deprecated so some Linux systems might not have it). Notice that the IP address of the network containers endpoint is not listed and no network device is listed for it either. That's because as far as the Linux kernel is concerned it doesn't exist.

What are you pinging? What is happening here?

The `zerotier-sdk-service` binary has joined a *virtual* network and is running a *virtual* TCP/IP stack entirely in user space. As far as your system is concerned it's just another program exchanging UDP packets with a few other hosts on the Internet and nothing out of the ordinary is happening at all. That's why you never had to type *sudo*. It didn't change anything on the host.

Now you can run an application inside your network container.

    export LD_PRELOAD=`pwd`/libzerotierintercept.so
    export ZT_NC_NETWORK=/tmp/netcon-test-home/nc_8056c2e21c000001
    node netcon/httpserver.js

Also note that the "pwd" in LD_PRELOAD assumes you are in the ZeroTier source root and have built netcon there. If not, substitute the full path to *libzerotierintercept.so*. If you want to remove those environment variables later, use "unset LD_PRELOAD" and "unset ZT_NC_NETWORK".

If you don't have node.js installed, an alternative test using python would be:

    python -m SimpleHTTPServer 80

If you are running Python 3, use `-m http.server`.

If all went well a small static HTTP server is now serving up the current directory, but only inside the network container. Going to port 80 on your machine won't work. To reach it, go to the other system where you joined the same network with a conventional ZeroTier instance and try:

    curl http://NETCON.INSTANCE.IP/

Replace *NETCON.INSTANCE.IP* with the IP address that *zerotier-netcon-service* was assigned on the virtual network. (This is the same IP you pinged in your first test.) If everything works, you should get back a copy of ZeroTier One's main README.md file.

## Installing in a Docker container (or any other container engine)

If it's not immediately obvious, installation into a Docker container is easy. Just install `zerotier-sdk-service`, `libztintercept.so`, and `liblwip.so` into the container at an appropriate locations. We suggest putting it all in `/var/lib/zerotier-one` since this is the default ZeroTier home and will eliminate the need to supply a path to any of ZeroTier's services or utilities. Then, in your Docker container entry point script launch the service with *-d* to run it in the background, set the appropriate environment variables as described above, and launch your container's main application.

The only bit of complexity is configuring which virtual network to join. ZeroTier's service automatically joins networks that have `.conf` files in `ZTHOME/networks.d` even if the `.conf` file is empty. So one way of doing this very easily is to add the following commands to your Dockerfile or container entry point script:

    mkdir -p /var/lib/zerotier-one/networks.d
    touch /var/lib/zerotier-one/networks.d/8056c2e21c000001.conf

Replace 8056c2e21c000001 with the network ID of the network you want your container to automatically join. It's also a good idea in your container's entry point script to add a small loop to wait until the container's instance of ZeroTier generates an identity and comes online. This could be something like:

    /var/lib/zerotier-one/zerotier-netcon-service -d
    while [ ! -f /var/lib/zerotier-one/identity.secret ]; do
      sleep 0.1
    done
    # zerotier-netcon-service is now running and has generated an identity

(Be sure you don't bundle the identity into the container, otherwise every container will try to be the same device and they will "fight" over the device's address.)

Now each new instance of your container will automatically join the specified network on startup. Authorizing the container on a private network still requires a manual authorization step either via the ZeroTier Central web UI or the API. We're working on some ideas to automate this via bearer token auth or similar since doing this manually or with scripts for large deployments is tedious.

## Tests

For info on testing the SDK, take a look at [docs/testing.md](docs/testing.md)

## Mobile App Embedding

For information on the app-embedding aspect of the SDK check out our [ZeroTier SDK](http://10.6.6.2/ZeroTier/ZeroTierSDK/docs/master/zt_sdk.md) blog post.

## Limitations and Compatibility

The beta version of the SDK **only supports IPv4**. There is no IPv6 support and no support for ICMP (or RAW sockets). That means network-containerizing *ping* won't work.

The virtual TCP/IP stack will respond to *incoming* ICMP ECHO requests, which means that you can ping it from another host on the same ZeroTier virtual network. This is useful for testing.

#### Controlling traffic

**Network Containers are currently all or nothing.** If engaged, the intercept library intercepts all network I/O calls and redirects them through the new path. A network-containerized application cannot communicate over the regular network connection of its host or container or with anything else except other hosts on its ZeroTier virtual LAN. Support for optional "fall-through" to the host IP stack for outgoing connections outside the virtual network and for gateway routes within the virtual network is planned. (It will be optional since in some cases total network isolation might be considered a nice security feature.)

The exception to this rule is if you use a network library in your application that supports the use of a SOCKS5 proxy and if you configure your network library to use the proxy service provided by the ZeroTier service you can disable all other shims and only talk to ZeroTier virtual networks via the proxied connections you specifically set up.

#### Compatibility Test Results

The following applications have been tested and confirmed to work for the beta release:

Fedora 23:

    httpstub.c
    nginx 1.8.0
    http 2.4.16, 2.4.17
    darkhttpd 1.11
    python 2.7.10 (python -m SimpleHTTPServer)
    python 3.4.3 (python -m http.server)
    redis 3.0.4
    node 6.0.0-pre
    sshd

CentOS 7:

    httpstub.c
    nginx 1.6.3
    httpd 2.4.6 (debug mode -X)
    darkhttpd 1.11
    node 4.2.2
    redis 2.8.19
    sshd

Ubuntu 14.04.3:

    httpstub.c
    nginx 1.4.6
    python 2.7.6 (python -m SimpleHTTPServer)
    python 3.4.0 (python -m http.server)
    node 5.2.0
    redis 2.8.4
    sshd

It is *likely* to work with other things but there are no guarantees.

