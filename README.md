#Tunlr Clone#
To build a tunlr or UnoTelly or unblock-us.com (or other DNS-based
services) clone on the cheap, you need to invest in a VPS.  For the
purposes of this discussion, I am assuming that you will be using
this for watching US geo-locked content.

##US IP Address##
Your VPS provider must provide you with a US IP address

##VPS Provider Specific Terminology##
My VPS provider is [buyvm](http://buyvm.net/).  I have an OpenVZ
128m plan with them hosted in New York (running Debian 7).  So, the
venet0 references that you will see pertain to that VPS provider.

##Disclaimer##
This information is provided as is without warranty of any kind,
either express or implied, including but not limited to the implied
warranties of merchantability and fitness for a particular purpose.
In no event shall the author be liable for any damages whatsoever
including direct, indirect, incidental consequential, loss of
business profits, or special damages.

If you leave your DNS server or HTTPS-SNI-Proxy server wide open to
abuse, that's on your own head.  Take precautions in this regard.
Proceed at your own risk.

Also this is not meant to be a hold-your-hand start from scratch
tutorial.  Therefore, some level of Linux expertise is necessary.

##Background##
Basically we are interested in proxying content only for certain
domains.  The actual streaming media sits on CDN networks and is
usually not geo-locked.  The amount of proxying we'll end up doing
will be relatively insignificant compared to a VPN-based setup.

[![How Tunlr Cloning works](https://raw.github.com/corporate-gadfly/Tunlr-Clone/master/tunlr-clone.png)](https://raw.github.com/corporate-gadfly/Tunlr-Clone/master/tunlr-clone.png)

User browses to Hulu homepage. Behind the scenes, this triggers the
following sequence of events:

1. Browsing device asks for the IP address of www.hulu.com (using DNS).
1. Since the router is running `dnsmasq`, it selectively sends the DNS
   query for www.hulu.com to DNS server running on the VPS.
1. The VPS DNS server responds with the IP address of VPS SNI Server as the
   authorative answer for the DNS query.
1. Router sends resolved IP address back to browsing device.
1. Browsing device sends a request for content for www.hulu.com.
1. VPS SNI Server sends a request for content to www.hulu.com.
1. Since the VPS SNI Server has an IP presence in USA, www.hulu.com
   responds with proper content.
1. VPS SNI Server proxies the content back to the browsing device

##Tomato based router##
Since you will be changing DNS servers to point to your "own" DNS,
it makes sense to run `dnsmasq` on your router, so that only relevant
DNS queries make it your DNS server and the vast majority of the
remaining DNS queries go to your regular ISP DNS.  Therefore having
a Tomato capable router is preferable (as Tomato has `dnsmasq`
capabilities).

Following is my `dnsmasq` configuration on my Tomato-based router
(running a Toastman build):
`Advanced -> DHCP/DNS -> Dnsmasq Custom configuration`
```bash
# Never forward plain names (without a dot or domain part)
domain-needed
# Never forward addresses in the non-routed address spaces.
bogus-priv

# If you don't want dnsmasq to read /etc/resolv.conf or any other
# file, getting its servers from this file instead (see below), then
# uncomment this.
no-resolv

# If you don't want dnsmasq to poll /etc/resolv.conf or other resolv
# files for changes and re-read them then uncomment this.
no-poll

# tunlr for hulu
server=/hulu.com/199.x.x.x
server=/huluim.com/199.x.x.x
server=/netflix.com/199.x.x.x
# tunlr for US networks
# cbs works with link.theplatform.com
server=/abc.com/abc.go.com/199.x.x.x
server=/fox.com/link.theplatform.com/199.x.x.x
server=/nbc.com/nbcuni.com/199.x.x.x
server=/pandora.com/199.x.x.x
server=/ip2location.com/199.x.x.x
# espn3 
server=/broadband.espn.go.com/199.x.x.x

# Google
server=8.8.8.8
server=8.8.4.4
# OpenDNS
#server=208.67.222.222
#server=208.67.220.220
```
`199.x.x.x` is the IP address of my VPS server (where my DNS server will
also be running). See next section.

In essence, I am forwarding DNS queries to my VPS only for the
specified domains.  Everything else goes to Google DNS (or can
easily go to your ISP DNS).

##Your own DNS Server##

I am running unbound on my VPS to override the DNS resolution for the domains
mentioned in the Tomato-based router configuration above. I used unbound because
it allows just overriding part of the domain, which allows media subdomains to
be forwarded to the original name servers. The plan is to send the external IP
address of my VPS as the resolved IP address for any of the overridden domains.

Once the web traffic hits my VPS, it will hit
[HTTPS-SNI-Proxy](https://github.com/dlundquist/HTTPS-SNI-Proxy)
running on port 80/443.

Here is the unbound config:

TODO: only netflix and pandora created so far.

`/etc/unbound/unbound.conf`
```
server:
    directory: "/etc/unbound"
    username: unbound
    chroot: "/etc/unbound"
    pidfile: "/etc/unbound/unbound.pid"
    interface: 0.0.0.0
    access-control: 199.195.255.68/31 allow

    local-zone: "netflix.com." transparent
    local-data: "netflix.com. IN A 199.x.x.x"
    local-data: "movies.netflix.com. IN A 199.x.x.x"
    local-data: "moviecontrol.netflix.com. IN A 199.x.x.x"
    local-data: "cbp-us.nccp.netflix.com. IN A 199.x.x.x"

    local-zone: "pandora.com." transparent
    local-data: "pandora.com. IN A 199.x.x.x"
    local-data: "www.pandora.com. IN A 199.x.x.x"
    local-data: "help.pandora.com. IN A 199.x.x.x"
    local-data: "blog.pandora.com. IN A 199.x.x.x"
```

##HTTPS-SNI-Proxy##
Install according to the instructions on
[HTTPS-SNI-Proxy](https://github.com/dlundquist/HTTPS-SNI-Proxy)

`/etc/sniproxy.conf`
```sniproxy.conf
# grep '^[^#]' /etc/sni_proxy.conf
user daemon
listener 172.y.y.y 80 {
    proto http
    table http
}
listener 172.y.y.y 443 {
    proto tls
    table https
}
table "http" {
    (hulu|huluim)\.com * 80
    abc\.(go\.)?com * 80
    (nbc|nbcuni)\.com * 80
    netflix\.com * 80
    ip2location\.com * 80
}
table "https" {
    (hulu|huluim)\.com * 443
    abc\.(go\.)?com * 443
    (nbc|nbcuni)\.com * 443
    netflix\.com * 443
    ip2location\.com * 443
}
```
##Firewall##

I used ufw to manage the firewall rules, it's simple. (I also allow SSH, so I'll
include it because if you forget you'll not be able to connect).

```bash
sudo apt-get install ufw
sudo ufw allow 22/tcp
sudo ufw allow from 199.195.255.68/31 to any port 53 proto udp
sudo ufw allow from 199.195.255.68/31 to any port 80 proto tcp
sudo ufw allow from 199.195.255.68/31 to any port 443 proto tcp
sudo ufw enable
```
