# FreeBSD pf gateway
## TLDR
This uses [[PF]] on [[FreeBSD]] to create a simple gateway between subnets
### FreeBSD gateway config
Edit rc.conf for boot config
``` shell
###/etc/rc.conf
ifconfig_vmx0="inet 192.168.2.23 255.255.255.0"
ifconfig_vmx1="inet 10.0.10.1 255.255.255.0"
static_routes="default"igroute_default="default 192.168.2.254"
gateway_enable="yes"
pf_enable="YES"
```
Edit pf.conf to config pf (firewall)
```shell
###/etc/pf.conf###
ext_if="vmx0"
int_if="vmx1"
localnet = $int_if:network

# ext_if IP address could be dynamic, hence ($ext_if)
nat on $ext_if from $localnet to any -> ($ext_if)
# Allow incoming SSH (port 22) traffic on the external interface
pass in on $ext_if proto tcp from any to any port 22
# Block all other incoming traffic by default
block in on $ext_if all

pass from { lo0, $localnet } to any keep state
```
Enable pf & reboot 
``` shell
sysrc pf_enable="YES"
service pf start
reboot
```
Add a default route
``` shell
route add default <external gateway IP>
```
### Debian client config
Edit network interfaces: /etc/network/interfaces
```shell
auto eth0 iface 
eth0 inet static 
	address 10.0.10.2 
	netmask 255.255.255.0 
	gateway 10.0.10.1
```
Restart networking
```shell
systemctl restart networking
```
Editing resolving: /etc/resolv.conf
```
#Can be any DNS server
nameserver 8.8.8.8 
nameserver 8.8.4.4
```
# Long version
Source: https://docs.freebsd.org/en/books/handbook/firewalls/

This section demonstrates how to configure a [[FreeBSD]] system running [[PF]] to act as a gateway for at least one other machine. The gateway needs at least two network interfaces, each connected to a separate network. In this example, vmx0 is connected to the Internet and vmx1 is connected to the internal network.

First, enable the gateway to let the machine forward the network traffic it receives on one interface to another interface. This sysctl setting will forward IPv4 packets:

```
# sysctl net.inet.ip.forwarding=1
```

To forward IPv6 traffic, use:

```
# sysctl net.inet6.ip6.forwarding=1
```

To enable these settings at system boot, use [sysrc(8)](https://man.freebsd.org/cgi/man.cgi?query=sysrc&sektion=8&format=html) to add them to /etc/rc.conf:

```
# sysrc gateway_enable=yes
# sysrc ipv6_gateway_enable=yes
```

Verify with `ifconfig` that both of the interfaces are up and running.

Next, create the PF rules to allow the gateway to pass traffic. While the following rule allows stateful traffic from hosts of the internal network to pass to the gateway, the `to` keyword does not guarantee passage all the way from source to destination:

pass in on xl1 from xl1:network to xl0:network port $ports keep state

That rule only lets the traffic pass in to the gateway on the internal interface. To let the packets go further, a matching rule is needed:

pass out on xl0 from xl1:network to xl0:network port $ports keep state

While these two rules will work, rules this specific are rarely needed. For a busy network admin, a readable ruleset is a safer ruleset. The remainder of this section demonstrates how to keep the rules as simple as possible for readability. For example, those two rules could be replaced with one rule:

pass from xl1:network to any port $ports keep state

The `interface:network` notation can be replaced with a macro to make the ruleset even more readable. For example, a `$localnet` macro could be defined as the network directly attached to the internal interface (`$xl1:network`). Alternatively, the definition of `$localnet` could be changed to an _IP address/netmask_ notation to denote a network, such as `192.168.100.1/24` for a subnet of private addresses.

If required, `$localnet` could even be defined as a list of networks. Whatever the specific needs, a sensible `$localnet` definition could be used in a typical pass rule as follows:

pass from $localnet to any port $ports keep state

The following sample ruleset allows all traffic initiated by machines on the internal network. It first defines two macros to represent the external and internal 3COM interfaces of the gateway.
``` shell
ext_if = "vmx0"	# macro for external interface - use tun0 for PPPoE
int_if = "vmx1"	# macro for internal interface
localnet = $int_if:network


# ext_if IP address could be dynamic, hence ($ext_if)
nat on $ext_if from $localnet to any -> ($ext_if)
block all
pass from { lo0, $localnet } to any keep state
```
This ruleset introduces the `nat` rule which is used to handle the network address translation from the non-routable addresses inside the internal network to the IP address assigned to the external interface. The parentheses surrounding the last part of the nat rule `($ext_if)` is included when the IP address of the external interface is dynamically assigned. It ensures that network traffic runs without serious interruptions even if the external IP address changes.

Note that this ruleset probably allows more traffic to pass out of the network than is needed. One reasonable setup could create this macro:

``` shell
client_out = "{ ftp-data, ftp, ssh, domain, pop3, auth, nntp, http, \
    https, cvspserver, 2628, 5999, 8000, 8080 }"
```

to use in the main pass rule:

``` shell
pass inet proto tcp from $localnet to any port $client_out \
    flags S/SA keep state
```

A few other pass rules may be needed. This one enables SSH on the external interface:

```shell
pass in inet proto tcp to $ext_if port ssh
```
This macro definition and rule allows DNS and NTP for internal clients:

``` shell
udp_services = "{ domain, ntp }"
pass quick inet proto { tcp, udp } to any port $udp_services keep state
```

Note the `quick` keyword in this rule. Since the ruleset consists of several rules, it is important to understand the relationships between the rules in a ruleset. Rules are evaluated from top to bottom, in the sequence they are written. For each packet or connection evaluated by PF, _the last matching rule_ in the ruleset is the one which is applied. However, when a packet matches a rule which contains the `quick` keyword, the rule processing stops and the packet is treated according to that rule. This is very useful when an exception to the general rules is needed.
## Troubleshooting
- no route to host on gateway -> `route add default [gateway IP]`