---
title: DVWA
---

# Damn Vicious Web Access

A phrase inspired from [DVWA](https://github.com/digininja/DVWA), but the current discussion has nothing to do with penetration testing. Recently, I had been suffering from immense frustration with my internet connectivity (fuck ISP! fuck!!), and only after quite a long time was I able to figure out what was the problem! An erroneous **NAT64 translation** at the ISP end, barring me from accessing certain IPv4-only sites (GitHub, Reddit, to name a few!). To get a comprehensive view of how an ISP provided IPv6-only (primarily) network works, one can refer [Jio 5G - IPv6 only on transport](https://anuragbhatia.com/post/2023/02/jio-5g-ipv6-only/).

## Workaround

The simplest one would be to use a functional DNS64/NAT64 translation. To understand how this works, help yourself:

- [DNS64 and NAT64 for 6to4 connectivity](https://docs.cloud.google.com/vpc/docs/ipv6-to-ipv4-overview)

- [Let Your IPv6-only Workloads Connect to IPv4 Services](https://aws.amazon.com/blogs/aws/let-your-ipv6-only-workloads-connect-to-ipv4-services/)

:clap: Hats off to the following public NAT64/DNS64 service providers:

- [level66](https://level66.services/services/nat64/)

- [nat64.net](https://nat64.net/)

You can have your own [BIND9](https://www.crc.id.au/2024/10/06/secure-dns-with-bind-and-dot/), with a grain of [DoT](https://unix.stackexchange.com/questions/735368/how-to-use-dns-over-tls-with-bind9-forwarders)/[DoH](https://medium.com/@kevintim/mastering-dns-privacy-configuring-dns-over-https-doh-and-dns-over-tls-dot-in-bind9-cc6db67aaab3)!

## The Hard Way!

For privacy concerns (hehe, boi!), and to "seemingly" convert the damn IPv6-only to IPv4-only stack, I decided to setup a VPN. Now, where to get a remote machine to host it, seriously! where?? I don't know why I chose [Google Cloud](https://cloud.google.com/), mostly because I wasn't able to create an [Oracle Cloud](https://www.oracle.com/in/cloud/free/) free tier account (still not able to figure out, like WTF is wrong with their [card verification system](https://www.reddit.com/r/oraclecloud/comments/1h0b1g8/help_unable_to_sign_up_for_oracle_cloud_free_tier/)? fuck!!), and I was looking for some kind of _always free_ quota! (sorry! [AWS](https://aws.amazon.com/), [Azure](https://azure.microsoft.com/en-us))

## Setup

- You need a virtual private network, right? so, learn to [Create and manage VPC networks](https://docs.cloud.google.com/vpc/docs/create-modify-vpc-networks), and for my case, I chose a combo of IPv4-only with IPv4/IPv6 (dual-stack, with an **external** IPv6 access type)

- Too rich? no?? know [How to Host Your Side Projects for $0: The Ultimate GCP Free Tier Guide](https://dev.to/jeaniscoding/how-to-host-your-side-projects-for-0-the-ultimate-gcp-free-tier-guide-3p07)

- 2 network cards, yeah! acquaint yourself with [Using a multi-nic VM as a gateway between VPCs in Google Cloud](https://medium.com/google-cloud/using-a-multi-nic-vm-to-connect-vpcs-in-google-cloud-d84aa533538) (pay special attention to `IP forwarding`!)

- Finally, WireGuard! I leave you with 3 choices:
  - [Docker-WireGuard](https://github.com/JustABeginning/Docker-WireGuard)

  - [Setting Up a WireGuard VPN Server on Google Cloud Platform](https://dev.to/gamedev90/setting-up-a-wireguard-vpn-server-on-google-cloud-platform-bc2) (**Note:** VPC Network &rarr; Firewall Rules, don't fuck up here!)

  - [Stupid simple setting up WireGuard - Server and multiple peers](https://gist.github.com/chrisswanda/88ade75fc463dcf964c6411d1e9b20f4) (especially, the `/32`-prefix in the server's peers' `AllowedIPs`)

## Fuck No!

Oh yeah! the hair tearing begins! Everything seems to be in track &ndash; VPC, Firewall, VM, `sysctl` (yes! just miss setting `net.ipv4.ip_forward = 1`, and see what a slicky ass MF it is!), WireGuard, everything configured, every syntax correct! and then, bam!! a hit right in the face! A refreshing ping from IPv4-only `nic0` and, a straight "fuck you!" from the dual-stack IPv6-address beholder `nic1`. Give up! right? since, at this point, what was even the point of doing all this shit! in the first place? Maybe, I would have ..., hadn't I stumbled across [x-yuri](https://serverfault.com/a/1145556)'s answer. So, with post-nut clarity, I began:

- Unclogging the IPv6 routes:
  - Look for the IPv6 gateway on `nic1` (`ens5`):

    ```bash

    #!/bin/bash

    ip a
    ip -6 route

    ```

  - Fix the path:

    ```bash

    #!/bin/bash

    sudo ip -6 route add default via <IPv6_gateway_on_ens5> dev ens5 table 1
    sudo ip -6 rule add from <IPv6_network_on_ens5> table 1

    ```

  - Verify:

    ```bash

    #!/bin/bash

    ip -6 rule show table 1
    ip -6 route show table 1

    ```

- **Optional:** Create a service (so that, you don't lose track with VM restart!)

  ```systemd

  [Unit]
  Description=Fix Routing
  After=network-online.target

  [Service]
  ExecStart=bash <path_to_shell_script>

  [Install]
  WantedBy=multi-user.target

  ```

## Multi-Layer

The first time I executed `warp-cli connect` with [WARP](https://developers.cloudflare.com/warp-client/get-started/linux/) (sounds more like, "Now from the top, make it drop, that's some WAP ...", hehe, just kidding!), I was kicked out of my own server! Had to delete and re-create the VM :pensive:, fuck!! An idiot enough not to realize that ["All device traffic is routed through WARP"](https://developers.cloudflare.com/warp-client/warp-modes/) rather than the standard local gateway. Now, as I was bent upon using [Zero Trust WARP](https://blog.cloudflare.com/zero-trust-warp-with-a-masque/), and with [split tunnels](https://www.reddit.com/r/CloudFlare/comments/14ucste/warp_on_ubuntu_vps_no_ssh_access/) being of no help! I proceeded with `warp-cli mode proxy` that establishes a tunnel for use in a SOCKS5 proxy over `127.0.0.1:40000`.

To utilize this mess (yeah! I still think there could have been a better way than a damn proxy! or, maybe I'm stupid!), I fired up a WireGuard interface (say, `wg1`), but how the fuck am I supposed to interface it with a proxy? [redsocks](https://forum.proxyrack.com/t/convert-a-socks-proxy-to-http-protocol-using-nginx-and-redsocks/68)? maybe, but for me? hell nah! Time to go nuts? almost! Not if you know [Redirecting All Container Traffic via SOCKS Proxy](https://bigmike.help/en/devops/003/) using [tun2socks](https://github.com/xjasonlyu/tun2socks/wiki/Examples) &ndash; an excellent technique of using `fwmark` with `iptables -t mangle`! And, as expected, I fucked up! forgetting to [allow forwarding](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/6/html/security_guide/sect-security_guide-firewalls-forward_and_nat_rules) for `wg1`:

```bash

#!/bin/bash

sudo iptables -A FORWARD -i wg1 -j ACCEPT

```

- Verify `ip rule` and `iptables`:

  ```bash

  #!/bin/bash

  ip rule show table <ROUTE_TABLE>
  ip route show table <ROUTE_TABLE>

  ```

  ```bash

  #!/bin/bash

  sudo iptables -t mangle -vnL PREROUTING --line-numbers
  sudo iptables -vnL FORWARD --line-numbers
  sudo iptables -t nat -vnL POSTROUTING --line-numbers
  sudo iptables-save

  ```

:metal: Ciao Adios!
