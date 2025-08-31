---
title: "Free VPNs Everywhere -- Tunnel Injection"
date: 2025-08-29T13:50:00+08:00
draft: false
github_link: ""
author: "Chumy"
tags:
  - Cyber Security
  - Networking
image: ""
description: ""
toc: 
---

# Tunnel Injection

[Chinese](/blogs/tunnel-injection)

## TL;DR
This blog discusses the flaws of most stateless tunnel protocols and how I turned them into a free VPN that can be set up with just 5 lines of Linux `ip` commands.

## Acknowledgments

Before getting into the main topic, I’d like to thank [seadog007](https://seadog007.work/) for [STUIX](https://stuix.io/), since my network’s ASN primarily peers through STUIX, and I also used a lot of their resources for testing.

Special thanks as well to [NCSE](https://ncse.tw/) for helping me by adding some IPT traffic during my testing.

## Introduction

This is a little story about how I went from researching a protocol to essentially giving the whole world access to my VPN.

Here’s how it happened: around my freshman year, I heard about this cool thing called Segment Routing v6 at a Cloud Native event. So, during my sophomore year, I started digging into it a bit and got a general understanding of its principles. As for how it actually works—since that’s not the focus of this post—I won’t go into detail here. I may publish another article on it in the future.

By junior year, I received an invitation from [2024 CGGC](https://cggc.nchc.org.tw/) (a CTF held in Taiwan) to contribute a challenge, which got me brainstorming for ideas.

![image](https://github.com/user-attachments/assets/15d204e5-72b9-4827-97a4-9abbcb51f621)

![image](https://github.com/user-attachments/assets/21be27d8-b269-47ee-8c1b-48f3831bea3d)

Then one day, while showering and replaying everything in my head, I suddenly came up with this really cool exploitation method. Fun fact: this particular challenge was only solved by [seadog007](https://seadog007.work/).

This is a detailed diagram of the attack method that [seadog007](https://seadog007.work/) drew for me.

![image](https://github.com/user-attachments/assets/ac66ca21-eadc-4000-98d9-6133a3f8456d)

## What is Tunnel

The tunnel most people are familiar with is the VPN, since a VPN is essentially just a type of tunnel. The core idea of a VPN is a virtual network built out of multiple encrypted or unencrypted tunnels.

Its working principle is actually quite simple—it’s basically the concept of putting an envelope inside another envelope.

![image](https://github.com/user-attachments/assets/6e5c692c-183f-41a2-a4d4-e03cb3eb4202)

![image](https://github.com/user-attachments/assets/c71b10e9-f6fa-473d-a94a-77e99d0a3dd5)

In networking terms: when a packet leaves the tunnel interface, the program handling the tunnel takes the contents of that packet, encapsulates it according to the tunnel protocol, and then sends it out via TCP, UDP, or some Layer 3 protocol payload—through either a physical NIC or another lower-layer tunnel interface.

On the receiving side, the program listening for this tunnel protocol takes incoming data from the listening port, strips off the tunnel protocol header, and delivers the inner raw data directly as a Layer 3 or Layer 2 packet to the tunnel interface.

![image](https://github.com/user-attachments/assets/88cf1d26-7f70-485e-8089-6e39ec9e4bca)

A stateless tunnel protocol simply means: whenever it receives a packet in that protocol format, it just decapsulates and forwards it—without performing any kind of verification or handshake.

## Impact

So what happens if there’s no verification? Basically, as long as I can forge a packet that matches the tunnel protocol format, I can inject malicious packets through this unprotected tunnel and deliver them into the victim’s internal network.

However, past research has generally considered this kind of attack to be one-way—at most enabling DDoS attacks.

![image](https://github.com/user-attachments/assets/9c57dd15-5db7-438b-9d39-d7871cd189aa)

![image](https://github.com/user-attachments/assets/21321374-4940-4520-9e9a-618ce5ccb06e)

Including a paper published just this January: [research](https://www.top10vpn.com/research/tunneling-protocol-vulnerability/)

![image](https://github.com/user-attachments/assets/359e4baa-1152-4fe7-9445-976b6a74733b)

That study covered many scanning methods for exposed stateless tunnels, as well as issues with certain router models not verifying source IPs. But its main focus was still on DDoS attack surfaces.

In reality, you only need to shift your perspective a little: we don’t actually need traffic to flow back and forth through the tunnel.

The real essence of the attack is simply being able to inject traffic into the tunnel. That’s why I’ve decided to call it Tunnel Injection.

### Tunnel Injection to Internal Network

The first exploitation method allows us to achieve Interactive Internal Network Access.

We begin by crafting a forged packet that conforms to the tunnel protocol.

- The outer layer:

  - Source IP = attacker’s public IP (or in some tunnel protocols, since source IP verification is performed, this might require IP spoofing).

  - Destination IP = victim tunnel’s public IP.

- The inner layer:

  - Source IP = <font color="red">attacker’s public IP</font> ← this is the core idea here.

  - Destination IP = <font color="red">the private IP of the victim’s internal machine</font> you want to target.

![image](https://github.com/user-attachments/assets/7a95b8c2-cf6c-4d50-9439-1a7ff17d9f52)

So what happens once we send it out?

When the packet reaches the victim’s router, it gets decapsulated. The router then forwards the inner packet according to its routing table, while leaving behind a conntrack entry for the forwarding record.

![image](https://github.com/user-attachments/assets/d0e5e518-0435-42b3-96c1-5529311bfd73)

The internal machine will then see a packet with:

src = <font color="red">public IP</font>, dst = victim’s internal machine's IP

Naturally, it replies with:

src = victim’s internal machine’s IP, dst = <font color="red">public IP</font>

![image](https://github.com/user-attachments/assets/33f5e5d2-2954-457b-bae5-e43ba2b12040)

When this reply reaches the victim’s router, because the destination is the public IP, it simply gets forwarded out via the default gateway.

One important note: most routers won’t SNAT “odd” packet types like TCP SYN/ACK or ICMP Echo Reply (pong). So when these reply packets leave the victim’s router, they are not NATed. In such cases, the source IP remains the private IP, which may cause issues in certain scenarios.

Finally, the attacker receives these responses from the victim’s internal network.

![image](https://github.com/user-attachments/assets/2a483ec0-2a7d-445a-a8cb-403f24bcd8bc)

This enables Arbitrary Interactive Internal Network Access.

<iframe width="560" height="315" src="https://www.youtube.com/embed/KSNkpPdzw8o?si=pE8cLkI-OTlZILsh" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

### Tunnel Injection to External Network

Before explaining the attack, let’s first look at how NAT is usually implemented—using Linux as an example.

On a Linux kernel–based router, NAT is handled with netfilter and conntrack. The common user-facing tools are iptables and nftables. Here we’ll take iptables as an example.

Normally, to enable NAT, you might use:

```bash
iptables -t nat -A POSTROUTING -o wan -s 192.168.0.0/16 -j MASQUERADE
```

This rule means: when a packet leaves the <font color="red">wan</font> interface, if its source IP matches 192.168.0.0/16, then perform NAT using the wan interface’s public IP as the new source IP.

But in reality, many router vendors take a shortcut and simply write:

```bash
iptables -t nat -A POSTROUTING -o wan -j MASQUERADE
```

This means: whenever a packet leaves the <font color="red">wan</font> interface, NAT it with the wan interface’s public IP as the source.

Notice what’s missing—the source IP check. That means if the source IP is already a public IP, it still gets NATed. After all, [RFC 1918](https://datatracker.ietf.org/doc/html/rfc1918) is just a definition; nothing stops you from using a public IP as if it were private.

Now, what happens if this router is also running an exposed tunnel?

The answer: we get a free proxy jump point.

In practice, all we need to do is modify the inner destination IP from the [Tunnel Injection to Internal Network](#tunnel-injection-to-internal-network) example—this time setting it to the target public IP on the external network.

Again, we forge a tunnel protocol–compliant packet:

- The outer layer:

  - Source IP = attacker’s public IP (or spoofed).

  - Destination IP = victim tunnel’s public IP.

- The inner layer:

  - Source IP = <font color="red">attacker’s public IP</font>

  - Destination IP = <font color="red">the target public IP you want to access</font>.

![image](https://github.com/user-attachments/assets/ffbfa772-3f79-4701-ad49-b968441ba406)

When the packet arrives at the victim’s router, it gets decapsulated, forwarded according to the routing table, NATed according to the `iptables` rule above, and logged into conntrack as a NAT entry.

![image](https://github.com/user-attachments/assets/b120f257-d8b2-4392-a619-80133643455c)

The target then receives the packet and replies. Since NAT has already rewritten the source IP, the target sees:

- src = <font color="red">victim public IP</font>, dst = <font color="red">target public ip</font>

So its response is:

- src = <font color="red">target public ip</font>, dst = <font color="red">victim public IP</font>

![image](https://github.com/user-attachments/assets/92a6f6f4-293e-42ff-80be-e659a6bd32cd)

When the victim’s router receives the response, conntrack restores the original destination IP and forwards it according to its routing table.

![image](https://github.com/user-attachments/assets/9ab444c2-9e01-4723-996e-e006786c340c)

And now the attacker receives the response.

![image](https://github.com/user-attachments/assets/9c24ec6f-6eae-4fe0-a7fd-df7588e38fd2)

This enables an Arbitrary Interactive Pivoting Attack via Tunnel.

![image](https://github.com/user-attachments/assets/6dfa50fa-0f75-46f5-acf2-b4d52f2a313c)

<iframe width="560" height="315" src="https://www.youtube.com/embed/EC9qySzo6J4?si=mF9rBsuFrUwao2-k" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## RPF or source ip verify bypass

Just when you think you’re done, reality often hits you in the face.

Remember what we mentioned earlier in [Tunnel Injection to Internal Network](#tunnel-injection-to-internal-network):

```
Most routers don’t perform SNAT on unusual packet types like TCP SYN/ACK or ICMP type 0 (pong).  
So when the response packet leaves the victim router, it won’t be NATed—meaning the source IP remains private.  
This can be problematic in certain scenarios.
```

And that “scenario” refers to Reverse Path Forwarding (RPF) or source IP verification.

![image](https://github.com/user-attachments/assets/e80bb09f-a306-4308-944b-938e24e01da0)

Most ISPs, in order to prevent customers from launching IP spoofing attacks, will enable RPF on client-facing interfaces.

RPF works like this: when an interface receives a packet, it checks the source IP against the routing table. If the source IP doesn’t match any route that should arrive via that interface, the packet is dropped.

So if the victim’s ISP enforces this protection, you won’t receive a response.
How do we bypass this?

### Bypass for Internal Network Access

Let’s take a look at how P2P networks implement NAT hole punching.

If both sides already know (or can predict) each other’s source ports—in the case of P2P applications, a STUN server usually helps clients discover their NATed source ports—then both sides can simultaneously send a packet to the other’s public IP and port. Let’s use UDP as an example.

![image](https://github.com/user-attachments/assets/590fc3b4-e990-4ebb-8f8d-e44c20f9f30f)

As the packets traverse the routers, NAT is applied and the conntrack table records a NAT mapping entry.

![image](https://github.com/user-attachments/assets/1a9e7d65-5fc1-44c0-bbed-92361eaed76d)

Both packets reach each other’s routers, where conntrack already has a matching NAT entry.

![image](https://github.com/user-attachments/assets/4e531fdf-9045-44aa-bb86-2b1b7189c7e2)

Thus, the destination IP is restored based on conntrack records.

![image](https://github.com/user-attachments/assets/368274e7-f142-4270-becf-f3589dfd908a)

That’s the basic flow of NAT hole punching. With TCP it’s more complex since TCP is stateful—look up TCP Simultaneous Open for details. In fact, HITCON CTF 2025 even had a [challenge](https://ctf2025.hitcon.org/dashboard/#19) based on this, since TCP Self-Connect is essentially a special case of TCP Simultaneous Open.

By the way, I only fully understood the details of P2P transmission because back in high school I [hand-coded the entire NAT hole punching process and even built my own STUN server](https://github.com/Jimmy01240397/NetworkServicePackage) (though looking back, the code is a bit ugly). Some people ask why reinvent the wheel, but I believe building it yourself is both fun and the best way to learn.

So we can think of NAT hole punching as a restricted form of port forwarding triggered from the inside.

Now—since Tunnel Injection allows us to inject packets from the inside out, could we manually perform NAT hole punching and then directly access the internal network from the outside?

Yes, we can. Let’s use TCP as an example.

We use Tunnel Injection to send a TCP SYN from within the victim’s internal network to perform hole punching.

- Source IP = victim internal machine’s <font color="red">private IP</font>
- Destination IP = attacker’s <font color="red">public IP</font>

![image](https://github.com/user-attachments/assets/34fa43d2-f10d-4c30-bf6b-9bba2537b88d)

The packet reaches the victim router, gets forwarded, NATed, and logged in conntrack.

Hole punched!

![image](https://github.com/user-attachments/assets/68ca12ef-176c-42c8-9306-406007a1eaa2)

Now we send a TCP SYN toward that hole:

- Source IP = <font color="red">attacker’s public IP</font>
- Destination IP = <font color="red">victim internal machine’s private IP</font>

![image](https://github.com/user-attachments/assets/6a05f51d-89e9-4b54-a0d8-a1af4b66f30e)

The router restores the destination IP via conntrack and forwards it to the victim’s internal machine.

![image](https://github.com/user-attachments/assets/9da739d2-8185-4c74-a558-1268905a25b5)

The victim machine responds with TCP SYN/ACK.

![image](https://github.com/user-attachments/assets/dd125103-65b5-4ff8-a28d-106f12453ba6)

And the source IP is NATed normally to the victim’s public IP.

![image](https://github.com/user-attachments/assets/dfc538ec-f8bd-4f92-ad21-d53ad41c4f77)

The attacker replies with ACK.

![image](https://github.com/user-attachments/assets/a2f3b7a5-61fd-45f8-a810-3076ed283495)

Now the connection is established, and the attacker can interactively communicate with the internal network.


![image](https://github.com/user-attachments/assets/30dec908-b538-4163-8151-7adc197584d2)

However, this process only works reliably for UDP.

For TCP, it only works on some routers or older Linux kernels. Modern Linux kernels perform strict conntrack validation of TCP Simultaneous Open. If one side never receives the SYN/ACK, the state machine remains stuck at SYN_SENT2. In this case, even if the internal host sends a TCP PUSH, <font color="red">it will not be NATed to the victim’s public IP</font>.

Since the attacker’s SYN/ACK never reaches the victim’s internal host, the connection never transitions to ESTABLISHED.

Could we solve this by manually sending the attacker’s SYN/ACK to the victim’s host?

Not really—because conntrack tears down the record whenever a TCP RST is seen, and most systems will respond to an unknown SYN/ACK with an immediate RST. (After all, the victim host never initiated the SYN, so it doesn’t recognize this session.)

While this trick may work in some networks where RST packets are dropped, we should assume the default case: it will fail.

So how to fix?

Looking at sysctl parameters, we find that conntrack’s TCP close state timeout is only 10 seconds. After that, the record is freed.

![image](https://github.com/user-attachments/assets/fcab59ab-27a7-4a7f-a1e5-10c82ae13728)

Another interesting parameter is `net.netfilter.nf_conntrack_tcp_loose`

![image](https://github.com/user-attachments/assets/84632096-8e77-4e61-bb89-35a1617bfbc3)

By default, this is set to 1.

What does it mean?

When `nf_conntrack_tcp_loose = 1`, conntrack allows NAT to be applied to stray TCP PUSH or ACK packets even if no matching session exists. In other words, when there is no active conntrack entry, an outbound TCP PUSH or ACK can create a new NAT entry in the ESTABLISHED state.

This opens up an opportunity: because the close-state timeout is short and TCP retransmits unacknowledged packets, we can:

1. Use Tunnel Injection to send a TCP RST from inside, freeing the conntrack record.
2. Wait for the victim host to retransmit its TCP PUSH.
3. Conntrack treats this retransmit as a new session, installs an ESTABLISHED NAT entry, and the attacker can now communicate interactively.

So here is the full flow recap

Use Tunnel Injection to send a TCP SYN from inside the victim’s network:

- Source IP = <font color="red">victim’s private IP</font>
- Destination IP = <font color="red">attacker’s public IP</font>

![image](https://github.com/user-attachments/assets/34fa43d2-f10d-4c30-bf6b-9bba2537b88d)

The packet reaches the victim router, gets forwarded, NATed, and logged in conntrack.

![image](https://github.com/user-attachments/assets/68ca12ef-176c-42c8-9306-406007a1eaa2)

Now we send a TCP SYN toward that hole:

- Source IP = <font color="red">attacker’s public IP</font>
- Destination IP = <font color="red">victim internal machine’s private IP</font>

![image](https://github.com/user-attachments/assets/6a05f51d-89e9-4b54-a0d8-a1af4b66f30e)

The router restores the destination IP via conntrack and forwards it to the victim’s internal machine.

![image](https://github.com/user-attachments/assets/9da739d2-8185-4c74-a558-1268905a25b5)

The victim machine responds with TCP SYN/ACK.

![image](https://github.com/user-attachments/assets/dd125103-65b5-4ff8-a28d-106f12453ba6)

And the source IP is NATed normally to the victim’s public IP.

![image](https://github.com/user-attachments/assets/dfc538ec-f8bd-4f92-ad21-d53ad41c4f77)

The attacker replies with ACK.

![image](https://github.com/user-attachments/assets/a2f3b7a5-61fd-45f8-a810-3076ed283495)

Attacker sends a PUSH (e.g., HTTP request).

![image](https://github.com/user-attachments/assets/3f55be71-accc-4b0e-8aa1-878ac136cc92)

Initially dropped. Victim retransmits repeatedly.

![image](https://github.com/user-attachments/assets/ef676fa9-b7d5-492b-86bd-ffd606b8c0bb)

Because of the mechanism explained above, the PUSH packet from the internal host is dropped. The attacker doesn’t receive it, doesn’t ACK it, and the internal host keeps retransmitting.

![image](https://github.com/user-attachments/assets/ef676fa9-b7d5-492b-86bd-ffd606b8c0bb)

We then use Tunnel Injection to send a TCP RST from inside, manually freeing the conntrack record.

![image](https://github.com/user-attachments/assets/7671e6c4-8041-4034-b9da-93fd323e0b1d)

After freeing it, when the internal host retransmits the TCP PUSH

![image](https://github.com/user-attachments/assets/115257d7-8133-4b60-9dd9-3177cbb7583d)

It will be successfully NATed, and conntrack will log a new NAT record in the <font color="red">ESTABLISHED state</font>.

![image](https://github.com/user-attachments/assets/b04c2350-c5b4-428f-b55d-7f665be6a19e)

Now the connection is fully established, and we can happily send data into the internal network.

![image](https://github.com/user-attachments/assets/30dec908-b538-4163-8151-7adc197584d2)

With this, we have successfully achieved RPF bypass for Internal Network Access.

<iframe width="560" height="315" src="https://www.youtube.com/embed/1YkltH1gCz4?si=V0iYDhGsGvKnd70O" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

### Bypass for External Network Access

The previous section was simpler because the entire bypass process only required a single NAT operation. However, for External Network Access, bypassing requires performing two NAT translations:

1. Changing the outbound packet’s source IP to the victim’s public IP
2. Changing the return packet’s source IP to the victim’s public IP

So, how do we achieve this?

If the victim’s internal network has another layer of tunnel combined with NAT, we can <font color="red">build a NAT Chain.</font>.

![image](https://github.com/user-attachments/assets/6e0361aa-f2ed-48ba-8bfa-b97eae5e9a5a)

We use Tunnel Injection to send a TCP SYN from the victim’s first-layer internal network to perform hole punching.

- Source IP = <font color="red">target’s public IP</font>
- Destination IP = <font color="red">attacker’s public IP</font>

![image](https://github.com/user-attachments/assets/eccd387e-42e7-44eb-8962-faa95ec0ffb2)

When the packet arrives at the victim’s first-layer router, it is forwarded according to the router’s routing table, NAT source IP translation is applied, and a conntrack entry is created.

![image](https://github.com/user-attachments/assets/9a6223c9-c748-41be-81c2-9ae450bdc4e7)

We then send a TCP SYN so that the first-layer router’s conntrack record transitions into the SYN_SENT2 state.

![image](https://github.com/user-attachments/assets/674aa743-e4b6-4bad-b05b-ea7763fb6f79)

This packet is dropped by RPF when forwarded to the target.

![image](https://github.com/user-attachments/assets/e8b0399a-96c1-4b5a-98be-a9f29b356cba)

Next, we use Tunnel Injection from the victim’s second-layer internal network to send a TCP SYN to the target.

- Source IP = <font color="red">attacker’s public IP</font>
- Destination IP = <font color="red">target’s public IP</font>

![image](https://github.com/user-attachments/assets/59105cb3-cdd6-4808-b561-499c8b839fcf)

![image](https://github.com/user-attachments/assets/0689a0c6-40e9-4ee9-b5f2-43aa9fed3f1a)

The TCP SYN is NATed by the second-layer router, and a conntrack entry is created.

![image](https://github.com/user-attachments/assets/00729446-faa7-401c-b282-030aae85a543)

It is then NATed again by the first-layer router, and a conntrack entry is created.

![image](https://github.com/user-attachments/assets/7be48293-0ae1-4f67-b4c1-bad4cae8c9d0)

The target responds with SYN/ACK, and the destination IP is restored according to conntrack.

![image](https://github.com/user-attachments/assets/61f1d773-6e80-4f93-a80f-eec9a253db2e)

The destination IP becomes the WAN IP of the second-layer router, which forwards it, restoring the destination IP according to its conntrack record.

![image](https://github.com/user-attachments/assets/0b813ca9-8604-48ad-9eb9-8f18c31d0416)

The destination IP then becomes the attacker’s public IP and is sent to the first-layer router.

![image](https://github.com/user-attachments/assets/a84a2fcb-a332-4680-8f34-2d07c5cfc5b3)

Here’s the interesting part: the source IP is the target’s public IP, which matches the NAT record that was previously created in the first-layer router.

So the source IP is NATed again according to conntrack.

![image](https://github.com/user-attachments/assets/48ecb285-68ef-4a9f-82aa-030b7e9d910a)

The attacker finally receives a packet with the source IP rewritten to the <font color="red">victim’s first-layer router’s public IP</font>.

![image](https://github.com/user-attachments/assets/511aede6-4882-4676-a2a8-2204684283d7)

From here, the process is the same as described earlier in [Bypass for Internal Network Access](#bypass-for-internal-network-access).

ACK is returned.

![image](https://github.com/user-attachments/assets/ac76ca3c-6eae-40fb-9aba-fa0a0864c138)

PUSH is sent.

![image](https://github.com/user-attachments/assets/32ab2ce8-6caa-482a-add0-144899e0d438)

RST is sent to free the conntrack record.

![image](https://github.com/user-attachments/assets/5f543174-edbe-4aa2-8aef-bf269000150d)

The target continues retransmitting.

![image](https://github.com/user-attachments/assets/94a00362-e077-46f6-8c43-0cc7cda68ab3)

It is successfully NATed, and conntrack now records an ESTABLISHED entry.

![image](https://github.com/user-attachments/assets/6c0dbb36-6cf7-4b89-b55d-56acf98acaa6)

Thus, we have successfully achieved RPF bypass for External Network Access.

<iframe width="560" height="315" src="https://www.youtube.com/embed/_GcIFKyjGmE?si=qWA0cVCgFotki8nm" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

Moreover, the use of a NAT Chain is not limited to bypassing RPF. It can also be used during Internal Network Access to bypass server-side source checks, since some servers also enforce their own firewall rules.

## L2 Tunnel Abuse

Earlier, we mainly talked about exploiting L3 tunnels. L2 tunnels are a bit more troublesome because they require an additional Ethernet header. This means we need the router’s or an internal machine’s MAC address to make use of them—or alternatively, we can attempt to exploit broadcast or multicast.

So, can we use broadcast or multicast to leak useful information such as MAC addresses?

Yes, it’s possible.

When an internal network not only uses IPv4 but also enables IPv6 with unicast addresses, ICMPv6 requests are allowed to be multicast by default. This means we can use multicast IPv6 ping to leak IPv6 addresses.

![image](https://github.com/user-attachments/assets/abc2edae-bf6e-46f9-ae66-001080bdffb2)

First, we craft a multicast ping inside the victim’s internal network, using a multicast MAC address and the attacker’s public IPv6 address.

![image](https://github.com/user-attachments/assets/ea7fd3e3-1d59-403d-a125-cbc0243b043e)

The switch will multicast this ping to all devices subscribed to that multicast MAC address.

![image](https://github.com/user-attachments/assets/5465ba53-bc01-486a-806f-8472c1c52097)

The internal victim devices will then reply with ICMP pong messages back to the attacker.

![image](https://github.com/user-attachments/assets/950c89a1-933d-4d69-a962-b042104c807c)

As a result, we can obtain all IPv6 addresses of devices in the internal network.

![image](https://github.com/user-attachments/assets/4ed1c850-90c0-4e26-824f-82a9a49e0f22)

<iframe width="560" height="315" src="https://www.youtube.com/embed/Eqb1dv2bPzk?si=gX9q73c6ubSatqVy" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

Now, what can we do with these IPv6 addresses?

According to [RFC 4291](https://datatracker.ietf.org/doc/html/rfc4291), one method of dynamically generating SLAAC IPv6 addresses is based on the host’s MAC address.

![image](https://github.com/user-attachments/assets/39dad163-bb73-420f-bced-ca58c684cb1c)

An IPv6 address generated in this way can be reversed back into the original MAC address. By default, Linux uses this method to generate IPv6 addresses.

![image](https://github.com/user-attachments/assets/9e120a86-764d-4558-992c-07732265b8e0)

In addition, another researcher from Trend Micro, [123ojp](https://github.com/123ojp), has also been researching this type of attack vector. He discovered even more efficient exploitation methods specifically targeting VXLAN, an L2 tunnel protocol. If you are interested, check out his talk at [DEFCON 33](https://defcon.org/html/defcon-33/dc-33-speakers.html#content_60316).

## IPSec

Some may assume that simply enabling IPSec is enough to protect against these kinds of attacks. However, in reality, if your IPSec is misconfigured, you can still be vulnerable—even with IPSec enabled.

In theory, once IPSec is enabled, all packets of the specified protocol should be intercepted by xfrm before leaving the NIC, encrypted, and then transmitted. Likewise, any raw packets of the specified protocol entering the NIC should be dropped.

However, xfrm has a special parameter called level. This parameter allows outbound traffic to be encrypted while still permitting raw inbound packets. Although this setting is not enabled by default, you never know what your IT team might do after struggling all day to configure IPSec without success—they might flip something they shouldn’t out of frustration.

![image](https://github.com/user-attachments/assets/58e4082d-c02c-4299-a859-05bf197ea631)

<iframe width="560" height="315" src="https://www.youtube.com/embed/y1ZlsGSu-RY?si=9klBHYJLEvnkw1aR" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

As a side note: on EdgeOS, even when the level attribute is set to its default value require, raw packets may still be allowed. This was discovered by my friend [zen](https://zenwen.tw/), the current network administrator of [NCKU CCNS](https://www.ccns.io/). However, this finding is still pending further confirmation.

## Scanning and Real World

Finally, let’s talk about scanning.

Although [this research](https://www.top10vpn.com/research/tunneling-protocol-vulnerability/) has already described concrete scanning methods, I’ll share my own approach here.

### Internal access able

We can test this by injecting an ICMP Echo Request into the victim’s tunnel, originating from inside the victim’s internal network:

- Source IP = private IP
- Destination IP = scanner’s IP

In the ICMP ping payload, we include some marker information along with the tunnel’s outer source IP and destination IP.

If we later receive the exact same ICMP ping on the WAN side, we can confirm that the victim’s router has an exposed tunnel for that protocol.

Then, by checking whether the source IP of the received packet is rewritten to a public IP, we can determine whether the victim router performs NAT.

![image](https://github.com/user-attachments/assets/5cf3582e-5570-4f27-aa24-9fa9d02f16b4)

![image](https://github.com/user-attachments/assets/e3347bbf-949f-4682-aca2-051b501daa9d)

![image](https://github.com/user-attachments/assets/10ca7e2d-3a10-493e-92f3-8574b1fa36e8)

Next, to check for RPF (Reverse Path Forwarding), we can inject an ICMP Echo Reply (pong) through the victim’s tunnel:

- Source IP = private IP
- Destination IP = scanner’s IP

As mentioned earlier in [Tunnel Injection to Internal Network](#tunnel-injection-to-internal-network), "Routers generally do not SNAT unusual packets such as TCP SYN/ACK or ICMP type 0 (pong)."

Therefore, packets leaving the victim router will definitely have a private source IP.

By simply checking whether the scanner receives the reply, we can tell whether the victim’s ISP enforces RPF.

![image](https://github.com/user-attachments/assets/6f3f4e5e-dee9-4929-9a93-edfb430a7fe2)

![image](https://github.com/user-attachments/assets/b4ba60ec-3f38-4086-979d-af123ce393e3)

### External access able

Since the set of externally accessible tunnels is necessarily a subset of the internally accessible tunnels with NAT, we can take that filtered IP list and scan it again.

The setup is the same, but this time we add a controlled target host.

We inject an ICMP Echo Request into the victim’s tunnel from its internal network:

- Source IP = scanner’s IP
- Destination IP = target host’s IP

![image](https://github.com/user-attachments/assets/11f911c5-5654-4810-a88a-9a79596a6c3b)

If the tunnel is externally accessible, the victim router should NAT the source IP to its public IP and forward the packet to the target. The target must be configured to log all incoming packets.

![image](https://github.com/user-attachments/assets/14d6f543-e252-49c7-ac93-ed754cd525ec)

The target will reply with an ICMP Echo Reply, which returns to the victim router.

![image](https://github.com/user-attachments/assets/aed43a57-e31d-4eda-bde2-7c56ba9c588b)

The victim router restores the destination IP and sends the packet back to the scanner.

At this point, by comparing the scanner logs with the target logs, we can determine exactly which tunnels are externally accessible.

![image](https://github.com/user-attachments/assets/dda1ce57-eae0-46a9-88bc-31241d4de968)

### Scan Results

Here are the results of scanning tunnels that do not require IP spoofing:

In total:

- GRE: 230,035 hosts
  - With NAT enabled: 191,961 hosts
    - Externally accessible: 22,875 hosts
- IPIP: 76,209 hosts
  - With NAT enabled: 23,851 hosts
    - Externally accessible: 5,438 hosts

Since some ISPs assign dynamic IPs, there may be slight inaccuracies in these numbers.

We also analyzed the ASNs of these IPs. As expected, a large portion belonged to Chinese IP ranges.

GRE: 

![image](https://github.com/user-attachments/assets/efb0aa63-43e5-45a5-bf00-0dbd50c8e38a)

IPIP:

![image](https://github.com/user-attachments/assets/a69480e1-7fdc-4045-9b6d-f33a749b5151)

## Conclusion

In the end, it was a bit unfortunate — because [123ojp](https://github.com/123ojp) happened to be working on the same research slightly earlier, I didn’t end up submitting this work to [HITCON 2025](https://hitcon.org/2025/zh-TW/).

That said, I still managed to present it as a HITCON Lightning Talk, where I shared this interesting finding and also introduced the [Tunnel Injection To External Network](#tunnel-injection-to-external-network) technique that [123ojp](https://github.com/123ojp) hadn’t mentioned or considered. In the end, it turned into a fun and engaging discussion.

Going forward, I might try to submit this research to other security conferences.

As a side note: although all of the demos above were implemented with custom hand-written tools, in theory — if you don’t need to perform [RPF Bypass](#rpf-or-source-ip-verify-bypass) — the same internal and external attacks can actually be executed with just <font color="red">five lines of Linux ip commands</font>. I’ll leave that as homework for you all. XXD

