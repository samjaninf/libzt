# libzt
*Embed ZeroTier directly into your app*
***

<a href="https://www.zerotier.com"><img src="artwork/ztapp100.png" align="left" hspace="20" vspace="6"></a>

**ZeroTier** makes it easy to securely connect devices, servers, cloud VMs, containers, and apps everywhere and manage them at scale. Now, with **libzt** you can bake this ability directly into your app or service using your preferred language or framework. We provide a BSD socket-like API to make the integration simple.

<hr>

[![irc](https://img.shields.io/badge/IRC-%23zerotier%20on%20freenode-orange.svg)](https://webchat.freenode.net/?channels=zerotier)

Pre-Built Binaries Here: [zerotier.com/download.shtml](https://zerotier.com/download.shtml?pk_campaign=github_ZeroTierSDK).

### Example

```
#include "libzt.h"

char *str = "welcome to the machine";
char *nwid = "c7cd7c9e1b0f52a2";

zts_simple_start("./zt", nwid);
if((fd = zts_socket(AF_INET, SOCK_STREAM, 0)) < 0) {
	printf("error creating ZeroTier socket");
}
if((err = zts_connect(fd, (const struct sockaddr *)addr, sizeof(addr))) < 0) {
	printf("error (%d) connecting to remote host\n", err);
}
int wrote = zts_write(fd, str, strlen(str));
zts_close(fd);
```

Bindings also exist for [many popular languages and platforms](examples).

***

All build targets will output to `build/`.

### Static Library

 - `make static_lib`

### iOS App Framework
 - `make ios_app_framework`

### macOS App Framework
 - `make macos_app_framework`

***

## Using Language Bindings
 - `SDK_LANG_JNI=1`: Enable JNI bindings for Java (produces a shared library)
 - `SDK_LANG_CSHARP=1`
 - `SDK_LANG_PYTHON=1`
 - `SDK_LANG_GO=1`

## Debugging flags
 - `SDK_DEBUG=1` - Enable SDK library debugging
 - `ZT_DEBUG=1` - Enable core ZeroTier service debugging

### Tests 
 - See [TESTING.md](TESTING.md)