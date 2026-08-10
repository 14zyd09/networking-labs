# VLANs, Trunking and Router-on-a-Stick

In this lab we build a simple two-floor office VLAN network starting up from a base flat, unsegmented switch and ending with isolated departments, extended across floors, and appropriately routed to each other.

The lab is built in Packet Tracer with 2960 Switches and a single 1941 Router-on-a-Stick

## Final Topology

With different departments in separate inter-Routable VLANs 

Physical:

![](https://i.imgur.com/xRCwr58.png)

Logical:

![](https://i.imgur.com/fTif7ov.png)

---

## 1. Flat Switched Network

All three departments (Sales, Marketing, HR) in one flat subnet (192.168.1.0/24). Here hosts can see traffic of different departments which is a security problem.

Physical topology:

![](https://i.imgur.com/Fp70oqe.png)

Logical topology:

![](https://i.imgur.com/pJObWfy.png)

Switch Port to Department Mapping:

| Port(s)     | Department |
|:----------- |:----------:|
| F0/1 - F0/6 | Sales      |
| F0/7 - F0/8 | Marketing  |
| F0/9        | HR         |

Any host can reach any other host: (We solve this in the next stage)

```
C:\>ping 192.168.1.6
Reply from 192.168.1.6: bytes=32 time<1ms TTL=128
```

---

## 2. VLANs - Isolated Departments

With VLANs we can easily segment the network from within our single switch

Logical Network diagram

![](https://i.imgur.com/uMHYnoq.png)

### **Config:**

First we **create** 3 VLANs for our departments and **assign** them to appropriate ports

```
Switch(config)#vlan 10
Switch(config-vlan)#name SALES
Switch(config-vlan)#exit
Switch(config)#
Switch(config)#interface range f0/1-6
Switch(config-if-range)#switchport mode access
Switch(config-if-range)#switchport access vlan 10
```

> The same pattern is repeated for Marketing VLAN 20, and HR VLAN 30

### **Good Practice: BLACKHOLE VLANs**

As a security measure we will disable connectivity on unused ports.

For this we will assign BLACKHOLE VLAN 100 and shutdown, all unused interfaces.

```
Switch(config)#vlan 100
Switch(config-vlan)#name BLACKHOLE
Switch(config-vlan)#exit
Switch(config)#
Switch(config)#int range f0/10-24
Switch(config-if-range)#switchport mode access
Switch(config-if-range)#switchport access vlan 100
Switch(config-if-range)#shutdown
```

### **Verify:**

In the same subnet, Sales can no longer reach HR or Merketing. But can reach other Sales.

```
C:\>ping 192.168.1.6
Reply from 192.168.1.6: bytes=32 time<1ms TTL=128

C:\>ping 192.168.1.10
Request timed out.
```

---

## 3. Trunks - Extending VLANs

A second floor has a matching VLAN configuration (Sales = 10; Marketing = 20) plus new Management VLAN 40. 

We want Floor2 to reach Floor1. And for this we need to set up a **Trunk Link** between our switches.

```
Switch(config)#interface f0/24
Switch(config-if)#switchport mode trunk
```

Updated Logical Topology:

![](https://i.imgur.com/5GSobtu.png)

### **Good Practice:**

To prevent the **VLAN Hopping** attacks we also need to change the default native VLAN from VLAN1 to any non-default VLAN.

```
Switch(config)#vlan 50
Switch(config-vlan)#name NATIVE
Switch(config-vlan)#exit
Switch(config)#interface f0/24
Switch(config-if)#switchport trunk native vlan 50
```

### **Verify:**

As a result same-department traffic reaches different floors. But different departments still can't reach each other:

Ping from Marketing A1 to Marketing B5 and Sales B2  

>  (192.168.1.11) (192.168.1.17) and (192.168.1.9) respectively

```
C:\>ping 192.168.1.17
Reply from 192.168.1.17: bytes=32 time<1ms TTL=128

C:\>ping 192.168.1.9
Request timed out.
```

---

## 4. Router-On-A-Stick - Routing between isolated VLANs

Now, for different departments to communicate to each other *(for business reasons)* we need to move each Department into separate subnets and configure a sub-interface on a router to give each VLAN a gateway 

New Subnets:

| Department | Subnet         | Gateway     |
| ---------- | -------------- | ----------- |
| Sales      | 192.168.1.0/24 | 192.168.1.1 |
| Marketing  | 192.168.2.0/24 | 192.168.2.1 |
| HR         | 192.168.3.0/24 | 192.168.3.1 |
| Management | 192.168.4.0/24 | 192.168.4.1 |

For each VLAN we set up individual sub-interfaces on G0/0.

```
Router(config)#interface g0/0.10
Router(config-subif)#encapsulation dot1Q 10 
```

And set the Gateway address for that VLAN subnet

```
Router(config-subif)#ip address 192.168.1.1 255.255.255.0
Router(config-subif)#no shutdown
```

LIkewise repeated for remaining Department VLANs

![](https://i.imgur.com/c4XNteg.png)

Plus another VLAN sub-interface for untagged traffic *(from legacy devices for example)*

```
Router(config)#interface g0/0.50
Router(config-subif)#encapsulation dot1q 50 native
Router(config-subif)#ip address 192.168.255.1 255.255.255.0
Router(config-subif)#no shutdown
```

### **Verify:**

Now all of our devices can reach each other.

> after the initial ARP resolution of course

Ping from Marketing A1 to Marketing B5 and Sales B2

> (192.168.2.2) (192.168.2.8) and (192.168.1.9) respectively

```
C:\>ping 192.168.2.8
Reply from 192.168.2.8: bytes=32 time<1ms TTL=128

C:\>ping 192.168.1.9
Request timed out.
Reply from 192.168.1.9: bytes=32 time=1ms TTL=127
```

---

# Troubleshooting

### Issue 1: Routed traffic not reaching any VLAN

**Symptom:**

After configuring all sub-interfaces with appropriate gateways, hosts still couldn't reach the gateway *(and hosts beyond it)*

**What i checked:** 

The router had correct gateway for each VLAN and sub-interfaces were up *(So it wasn't the router)*. 

Then I checked the switch interface connecting to the router *(Packet Tracer was showing a yellow dot)*

**Root Cause:** 

The switch interface connected to the router was set to **BLACKHOLE Access VLAN** set earlier. And **Access ports don't tag traffic**. Meaning the switch silently dropped the 802.1Q packets.

**Fix:** 

Set switch port facing the router to be a Trunk

```
Switch(config)#interface f0/23
Switch(config-if)#switchport mode trunk
Switch(config-if)#switchport trunk native vlan 50
```

**Lesson:** Router-on-a-stick only works if the switch port facing it is a Trunk.

## Issue 2: Some VLANs unreachable across floors

**Symptom:** HR *(VLAN 30, exists only on Floor 1)* couldn't reach Management *(VLAN 40, exists only on Floor 2)*.

**Root cause:** 

Because Floor 1 switch had no concept of VLAN 40 of Management and Floor 2 switch had no concept of VLAN 30 of HR, packets couldn't reach these two VLANs.

**Fix:**

Add the missing VLANS 

> No need to assign them to ports

On Floor 1 Switch:

```
Switch(config)#vlan 40
Switch(config-vlan)#name MANAGEMENT
```

On Floor 2 Switch:

```
Switch(config)#vlan 30
Switch(config-vlan)#name HR
```

**Lesson:** 

The Trunk can only carry packets for VLANs it knows about from it's VLAN database.

## **Final verification**

Traceroute from HR to Management shows that traffic goes through the expected path: HR Host > logical gateway *(192.168.3.1)* > Router > Management Host *(192.168.4.2)*

```
C:\>tracert 192.168.4.2

Tracing route to 192.168.4.2 over a maximum of 30 hops: 

  1   0 ms      0 ms      0 ms      192.168.3.1
  2   0 ms      0 ms      0 ms      192.168.4.2

Trace complete.
```

## Wrap-up

We started from a **Flat network**, isolated departments via **VLANs**, extended VLANs across switches by **Trunking**, restored inter-VLAN routing via **Router-on-a-Stick**. 

And Fixed two problems along the way (Port in access mode connected to the Router; Missing VLAN database entries).
















