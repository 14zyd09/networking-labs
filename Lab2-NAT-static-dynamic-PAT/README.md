# NAT (Static, Dynamic, PAT)

### The problem:

Every modern computer, phone, TV and IoT device needs an IP address in order to communicate with one another. 

But due to the shortage of available IPv4 addresses not every device can get a public IP. 

NAT (Network Address Translation) helps alleviate this problem by reusing a single IP address for multiple hosts.

In this lab we will take a network with a range of public IPs assigned by ISP and enable all devices to reach the internet.

<final topology diagram>

---

## Static NAT

We are given a network with multiple **consumer end devices** a single **Server**. 

And an IP address range of **(203.0.115.40/29)** with 6 usable addresses. provided by the ISP. 

![](images\static-nat.png)

The server needs a predictable address for inbound traffic so we enable a static translation to one of our public IPs.

#### Config:

We set the interface facing the ISP as outside NAT.

And the interface facing our network as inside NAT.

```
Router(config)#int g0/0
Router(config-if)#ip nat outside
Router(config-if)#int g0/1
Router(config-if)#ip nat inside
```

And set a simple static / direct translation between two IPs

```
Router(config)#ip nat inside source static 172.16.0.2 203.0.115.43
```

#### Verify

Now traffic goes from and to our local server from a remote server

```
C:\>ping 203.0.115.43
Request timed out.
Request timed out.
Reply from 203.0.115.43: bytes=32 time<1ms TTL=126
Reply from 203.0.115.43: bytes=32 time<1ms TTL=126
```

And IP address gets correctly translated:

```
Router#show ip nat translation
Pro  Inside global     Inside local       Outside local      Outside global
---  203.0.115.43      172.16.0.2         ---                ---
```

---

## Dynamic NAT

Static NAT translation is a predictable approach that works well for servers or networking equipment

But manually configuring translations doesn't scale well for many end hosts. 

With **Dynamic NAT** we can create a shared **pool of IP** addresses that automatically get assigned to hosts requiring a connection.

At the cost of not beeing able to initialize connections from outside networks

### config:

We create a pool with the remaining 3 IP addresses

```
Router(config)#ip nat pool LOCAL 203.0.115.44 203.0.115.46 netmask 255.255.255.248
```

And define an ACL with all of our hosts eligible for the translation

```
Router(config)#access-list 1 permit 172.16.0.0 0.15.255.255
```

And define the dynamic NAT

```
Router(config)#ip nat inside source list 1 pool LOCAL
```

### verify:

Our end hosts can reach the remote server

```
C:\>ping 157.15.0.95
Request timed out.
Reply from 157.15.0.95: bytes=32 time<1ms TTL=126
```

And IP addresses get translated

```
Router#show ip nat translation
Pro  Inside global     Inside local       Outside local      Outside global
icmp 203.0.115.44:1    172.16.0.6:1       157.15.0.95:1      157.15.0.95:1
icmp 203.0.115.44:2    172.16.0.6:2       157.15.0.95:2      157.15.0.95:2
icmp 203.0.115.44:3    172.16.0.6:3       157.15.0.95:3      157.15.0.95:3
icmp 203.0.115.44:4    172.16.0.6:4       157.15.0.95:4      157.15.0.95:4
icmp 203.0.115.45:10   172.16.0.3:10      157.15.0.95:10     157.15.0.95:10
icmp 203.0.115.45:11   172.16.0.3:11      157.15.0.95:11     157.15.0.95:11
icmp 203.0.115.45:12   172.16.0.3:12      157.15.0.95:12     157.15.0.95:12
icmp 203.0.115.45:9    172.16.0.3:9       157.15.0.95:9      157.15.0.95:9
---  203.0.115.43      172.16.0.2         ---                ---
```

And after a short time interval get released back into the pool:

```
Router#show ip nat translation
Pro  Inside global     Inside local       Outside local      Outside global
---  203.0.115.43      172.16.0.2         ---                ---
```

---

## PAT (Port Address Translation)

With Dynamic NAT hosts initiating outbound connections automatically get an IP. 

But if our available IP public address pool runs out, the remaining hosts won't be able to get an IP to reach the ISP.

PAT solves this by reusing a single IP between many hosts by differentiating translations by port numbers. 

### config:

We can simply add "overload" at the end of our dynamic NAT configuration

> Router(config)#ip nat inside source list 1 pool LOCAL **overload**

But as a good practice we can also reuse the global IP of our router.

> *Because we physically can't have new unsolicited inbound traffic anyway*

And to avoid having to reconfigure NAT every time the outside IP of our router changes. We can set the translation to the outside router interface.

```
Router(config)#ip nat inside source list 1 interface g0/0 overload
```

### verify:

Again the hosts can reach the outside network.

However, time all requests get translated into the IP of our routers outside interface

```
Router#sh ip nat translations
Pro  Inside global     Inside local       Outside local      Outside global
icmp 203.0.115.42:10   172.16.0.6:10      157.15.0.95:10     157.15.0.95:10
...
icmp 203.0.115.42:9    172.16.0.6:9       157.15.0.95:9      157.15.0.95:9
...
---  203.0.115.43      172.16.0.2         ---                ---
```

---

## Troubleshooting

Unfortunately no problems were encountered to exercise in troubleshooting.

But for examples sake lets consider what we would experience if one of our end hosts had a misconfigured IP address from a different subnet:

IP: 172.32.0.5/12

#### symptoms:

All requests to time out at the router:

```
C:\>tracert 157.15.0.95
  1   *         *         *         Request timed out.
```

Enabling debug mode for NAT translations `debug ip nat` doesn't show any output when attempting to reach the outside server.

And the misconfigured host cannot reach the gateway.

#### lessons:

NAT cannot tell us anything about the problem if no reachability to a host is established first. 

---

## Final verification

We can finally trace the route of a packet to verify that the outbound traffic from our end hosts travels the expected path:

> Router > ISP > Outside Server

```
C:\>tracert 157.15.0.95
  1   0 ms      1 ms      0 ms      172.16.0.1
  2   0 ms      0 ms      0 ms      203.0.115.41
  3   1 ms      0 ms      0 ms      157.15.0.95
```

---

## Wrap-up

In this lab we successfully configured static NAT for predictable server addressing, dynamic NAT for end host reachability to the ISP, PAT for public IP address conservation.

Unfortunately no problems were encountered during the lab to perform troubleshooting.
