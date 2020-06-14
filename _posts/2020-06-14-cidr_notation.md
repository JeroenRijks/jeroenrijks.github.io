---
layout: post
title: "CIDR notation"
subtitle: "What I learnt by creating AWS VPCs and subnets"
date: 2020-06-14 10:00:13 -0400
background: '/img/posts/09.jpg'
---

When I started working at Transreport, I was working with existing infrastructure, which other people had set in place for me. This let me get by without exposing me to a huge gap in my knowledge - networks, and how they work.

When creating VPCs in AWS, users need to input the VPC's CIDR block, which is a range of IP addresses, expressed (for IPv4 IP addresses) as `a.b.c.d/x`, where `a`, `b`, `c` and `d` are integers between 0 and 255, and `x` is an integer between 1 and 32. This is because IPv4 IP addresses can be written in binary as four 8-digit binary numbers: `aaaaaaaa.bbbbbbbb.cccccccc.dddddddd`, with `x` denoting how many of the 32 digits describe the network's address. Therefore, `32 - x` describes how many digits are dedicated to IP addresses inside this network. For example, the `10.1.0.0/16` CIDR block can be translated to `00001010.00000001.xxxxxxxx.xxxxxxxx`

As a rule, the `32 - x` binary digits are set as zeros in CIDR notation, which is why a `10.1.1.0/16` CIDR block would be impossible.

### Network sizes
As you may have noticed, smaller `x` values give larger networks, because a network has `2^(32-x)` unique numbers inside it, each mapping to one IP address.

### Subnets
Once I had figured this out, and created a VPC, I found my next problem - subnets. Subnets are subdivisions of the network they belong to, used to break networks down into smaller logical components, each dedicated to their own purpose, like hosting the database, running an API, etc. This can be useful for a host of reasons:

##### Security
With an understanding of which components communicate with which other components, subnets allowed me to set more fine-grained control over the flow of information inside my network. If the database has no reason to interact with our queueing system, then the database subnet can be set to not receive incoming or transmit outgoing traffic from the queue's subnet.

##### Improve network performance
Given that all devices in a network has an entry point into it, each device must dedicate resources to filtering to only respond to requests meant for itself. Subnets have a more limited scope than their parent network, meaning that the total incoming traffic is reduced for each device, with an effect spread across all network devices.

##### An understanding of the network's growth
Controlling the number of hosts available to each infrastructure component may be useful, in order to assign IP addresses to all your hosts necessary. You can dedicate a certain number of hosts to each subnet (depending on the size of the subnet mask), and leave room for each subnet to grow in the future.


### How subnets work
If you want to split a `10.1.0.0/16` network into subnets, you essentially only have `32 - x` binary digits to allocate to these subnets, which in this case is `16`: `00000000.00000000`. Just like with networks themselves, some of these digits will be dedicated to describing the subnet's address, and the rest will be dedicated to the IP addresses in this subnet. Large subnets need many IP addresses, leaving fewer available for the subnet's address. For example, if my subnet needs a maximum of 4 hosts, I only need 3 digits for that, so 13 can be dedicated to the subnet's address, which could range between `10.1.0.0/3` (`00001010.00000001.00000000.00000xxx`), and `10.1.255.248/3` (`00001010.00000001.11111111.11111xxx`). Careful decisions around subnet addresses and subnet sizes allow users to "future-proof" their networks, allowing for scaling of a subnet, without other subnets getting in the way.
For a given subnet, its size is conventionally described using its "subnet mask", which is the number you would get if all network address values were 1s. Therefore, `10.1.255.248/3` would give 255.255.255.248

I used Terraform's [cidrsubnet function](https://www.terraform.io/docs/configuration/functions/cidrsubnet.html) to define my subnets. This function takes three arguments: 
- `prefix`, the network's CIDR block
- `newbits`, the number by which to extend the prefix. The bigger this is, the smaller the subnet!
- `netnum`, which determines the value of the newbits (the subnet's address)

### Individual IP addresses`in CIDR notation
When I needed to temporarily whitelist an IP address, I wasn't sure how to do that, because AWS security groups and NACLs require inputs to use CIDR notation. The solution was very simple; as the number after the `/` gets larger, the range of IP addresses gets smaller. Therefore, to limit this range to a single IP address, simply add the `/32` suffix to the IP address, and your IP address will be converted into CIDR notation!

### Conclusion
I still have lots to learn on networking, so if you identify any holes in my knowledge, or would like to start a conversation about this with me, then please reach out to me by email.