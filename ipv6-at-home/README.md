# Home IPv6 access when your carrier is stuck in the IPv4-only age

## The backstory

I've been a FiOS Internet customer since we bought our current house about 12 years ago. Back then, everything was an Ethernet handoff, and you were given a crappy D-Link router. I replaced that in a hurry, naturally. As the years marched on, there were whispers everywhere about IPv6 and how it was the key to the survival of the universe. Some SPs came out swinging, and implemented v6 on their networks, with varying degrees of pain for you, the customer.

Then there were those other SPs who said stuff like, "Yep, we're looking at it, recognize it's the future, blah blah blah. We're totally going to implement it soon!" (I'm looking at you, Verizon.) Of course, for many of those, it still hasn't materialized. Eventually, I grew tired of waiting, and so I signed up for Hurricane Electric's free [Tunnel Broker service](https://www.tunnelbroker.net).

The key points on getting a tunnel from HE:

*   You will be issued a static /48 block of IPv6 address space.
*   An additional /64 will be used for the numbered IP-IP tunnel link (type is 6in4).
*   When you setup your tunnel, they provide sample configs for several different device types. Their samples for Junos are *very* old, and out of date.
*   Your WAN IP needs to be reachable from the outside by ping.

## Setup on SRX

### Enabling IPv6 Flow Mode

Before we start, it's worth noting that I'm using an SRX300, running Junos 15.1X49-D130. You'll need to change the SRX's v6 forwarding mode to flow-mode. This is achieved by:

```
set security forwarding-options family inet6 mode flow-based
commit
```

Once you commit that config, you'll be informed that you just changed the v6 forwarding mode type, and you'll need to reboot right now. So, go ahead and `request system reboot`, and then you'll continue.

### Setting up the tunnel interface

Now we're getting down to it - you'll be online with v6 soon. You'll need to configure an IPIP tunnel. You'll need to know your WAN v4 address, HE's tunnel endpoint's v4 address, and your v6 tunnel address (from the /64 they assign to the tunnel). Here's what mine looks like:

```
interfaces {
    ip-0/0/0 {
        unit 0 {
            tunnel {
                source <My WAN IPv4 address>;
                destination <HE Tunnel Endpoint IPv4 address>;
                path-mtu-discovery;
            }
            family inet6 {
                address <My IPv6 tunnel interface IP>/64;
            }
        }
    }
}
```

### Setting up Security Zone & Policies

You'll also need to setup some stuff in your security zones. You may not even need to fiddle with the WAN IFL, but I'll show it here just so you can see what it looks like. Obviously, I'm using ge-0/0/5.0 as my WAN-facing IFL.

```
security {
    security-zone untrust {
        interfaces {
            ge-0/0/5.0 {
                host-inbound-traffic {
                    system-services {
                        dhcp;
                        ike;
                        ping;
                    }
                }
            }
            ip-0/0/0.0 {
                host-inbound-traffic {
                    system-services {
                        ping;
                    }
                }
            }
        }
    }
}
```

As for Security Policy, chances are you're using an "anything outbound" policy, and not allowing inbound, or selectively allowing inbound traffic. Policies can also reference v4 and v6 addresses in the same or different policy rules.

### Outbound Routing for v6 Traffic

Next up, you'll need a default v6 route.  Easily done:

```
routing-options {
    rib inet6.0 {
        static {
            route ::/0 next-hop <HE's IPv6 tunnel link IP>;
        }
    }
}
```

At this point, you should be able to ping v6 addresses on the Internet. For example:
```
user@srx> ping www.google.com
PING6(56=40+8+8 bytes) <My v6 Tunnel IP> --> 2607:f8b0:4006:80e::2004
16 bytes from 2607:f8b0:4006:80e::2004, icmp_seq=0 hlim=58 time=13.045 ms
16 bytes from 2607:f8b0:4006:80e::2004, icmp_seq=1 hlim=58 time=12.864 ms
16 bytes from 2607:f8b0:4006:80e::2004, icmp_seq=2 hlim=58 time=50.167 ms
```

### LAN-Side Addressing

My internal interface is ae0.0, perhaps you're using integrated switching, and so yours may be irb.0 or maybe even something else. Only you can know what's right for your setup here. Now you can put a v6 address on your internal interface. First, let's talk about how you might number.

Suppose you were given 2001:1111:2222::/48, and have elected to use 2001:1111:2222::/64 as your internal network, assigning 2001:1111:2222::1 to the SRX's internal IFL. That might look like this:

```
interfaces {
    ae0 {
        aggregated-ether-options {
            lacp {
                active;
                periodic fast;
            }
        }
        unit 0 {
            family inet {
                address <My Internal IPv4 Address>/24;
            }
            family inet6 {
                address 2001:1111:2222::1/64;
            }
        }
    }
}
```

### Address Distribution

You can go 2 ways here - DHCPv6 or Stateless Address Autoconfig (aka SLAAC). In an environment outside your house, you probably want to go with DHCPv6. My example uses SLAAC. This is configured under protocols > router-advertisement.

```
protocols {
    router-advertisement {
        interface ae0.0 {
            prefix 2001:1111:2222::/64;
        }
    }
}
```

At this point, if you've got Windows in its default state, you should have a v6 address on your network interface. If you're on macOS, you can set your interface to configure IPv6 automatically.

## Automated Maintenance of Tunnel Endpoint Addressing

So, if you're like me with a dynamic IP WAN IFL, you'll need to maintain your end of the tunnel. I wrote a [Python script](https://github.com/jcostom/pyez-toys/blob/master/update-srx-ipip/update-srx-ipip.py) that uses the Junos PyEZ API to update the local end of the ip-0/0/0.0 IFL, as well as what HE expects to connect to by way of their HTTPS-based API.
