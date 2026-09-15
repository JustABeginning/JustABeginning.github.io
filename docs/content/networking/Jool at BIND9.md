---
title: BJ
date: 2026-08-15
update: 2026-10-10
---

**Published:** {{ page.date | date: "%a, %b %d, %Y" }}

**Modified:** {{ page.update | date: "%a, %b %d, %Y" }}

:hourglass: {{ page.content | number_of_words | divided_by: 200 | at_least: 1 }} min read

# Jool at BIND9

And not, drool at 69! (hehe, boi!) Guess what? I finally did it! For the "first" time, got my hands dirty setting up a functional DNS64/NAT64 translation in a "futile" attempt to build a **forwarding** DNS server with support for DoT/DoH. But, all of this! why? my dear ISP! (mut&#772;h&#601;fu\*\*er!!) So, buckle up, as I shit over here!

## NAT64

Let's get the "translation" done right! For quite sometime, I had to scratch my head really hard to figure it out! Like, how the fuck on earth can one get the damn [WKP](https://oneuptime.com/blog/post/2026-03-20-well-known-prefix-nat64-96/view) (`64:ff9b::/96`) working from a custom setup?

- Add a [static route](https://nicmx.github.io/Jool/en/run-nat64.html) (`ip route add 64:ff9b::/96 via 2001:db8::1`) from every IPv6-only client machine?

- Run it in a [BGP](https://blog.thelifeofkenneth.com/2019/02/running-nat64-in-bgp-environment.html) environment? ("NAT64 is a pretty clever way to add IPv4 connectivity to an IPv6-only network, since it takes advantage of the fact that the entire IPv4 address space can actually be encoded in half of a single subnet of IPv6", the best explanation ever :clap:!)

- Deploy at [ISP](https://oneuptime.com/blog/post/2026-03-20-deploy-nat64-dns64-isp-scale/view) scale? (sadly, I don't own my ISP!)

To quote [level66](https://level66.services/services/nat64/), "To be able to use the translation mechanism across multiple networks, we are using a IPv6 Global Unicast prefix as the NAT64 prefix", I realized the need to use an [NSP](https://www.infoblox.com/blog/ipv6-coe/nat64-well-known-prefix-vs-network-specific-prefix/), and with [Google Cloud](https://docs.cloud.google.com/vpc/docs/ip-addresses), I can get a "publicly advertised" **External IP** address; even better, "A `/96` is allocated to the VM on creation" when using [Global Unicast Addresses (GUA)](https://cloud.google.com/blog/products/networking/how-to-create-a-ipv6-network-in-google-cloud) &ndash; hell yeah!

### Break a Sweat!

VPC and VM configurations are similar to that of [DVWA](</Damn Vicious Web Access>), i.e., an `IP forwarding` set combo of IPv4-only with IPv4/IPv6 (dual-stack, with an **external** IPv6 access type). Time to hangout with `jool`!

- **Install:** Consulting the [official](https://nicmx.github.io/Jool/en/debian.html) documentation alongside [How to Configure NAT64 with Jool on Linux](https://oneuptime.com/blog/post/2026-03-20-configure-nat64-jool-linux/view), this seemed to work:

  ```bash

  #!/bin/bash

  sudo apt install linux-headers-$(uname -r)
  sudo apt update && sudo apt install jool-dkms jool-tools
  # Load the Jool NAT64 kernel module
  sudo modprobe jool
  # To load automatically at boot
  echo "jool" | sudo tee -a /etc/modules-load.d/jool.conf
  # Verify the module is loaded
  lsmod | grep jool

  ```

- **Pool:** First, know your `/96`-prefix allocated to `nic1` (`ens5`) as `inet6 <IPv6_address_on_ens5>/128 scope global dynamic noprefixroute` via:

  ```bash
  #!/bin/bash

  ip a

  ```

  - Next, on my way to [set up](https://oneuptime.com/blog/post/2026-01-15-nat64-dns64-ipv6-only-networks/view), I was greeted with `Error: Netfilter is the only available instance framework.`, the moment I ran:

    ```bash

    #!/bin/bash

    sudo jool instance add "nat64" --iptables --pool6 <IPv6_address_on_ens5>/96

    ```

    Wow! looks like, [netfilter](https://cooperlees.com/2020/12/nat64-using-jool-on-ubuntu-20-04/) was the only choice! However, since **Netfilter** is ["greedy"](https://nicmx.github.io/Jool/en/intro-jool.html) in that, _"Jool attempts to translate everything, and the rest of the network subsystem gets the leftovers"_, I was relieved of the burden to configure `iptables`/`ip6tables` to direct traffic to Jool, phew :relieved:!

    ```bash

    #!/bin/bash

    sudo jool instance add "nat64" --netfilter <IPv6_address_on_ens5>/96
    # Verify
    sudo jool -i nat64 global display | grep pool6

    ```

  - Moving on to `pool4` &ndash; Jool's [IPv4 transport address pool](https://nicmx.github.io/Jool/en/usr-flags-pool4.html), I felt an itch to use [tun2socks](https://github.com/xjasonlyu/tun2socks/wiki/Examples) with [WARP](https://developers.cloudflare.com/warp-client/get-started/linux/) instead of sacrificing the IPv4 address on `ens4` (`nic0`). Adopted everything but the `fwmark` and WireGuard stuff from [DVWA](</Damn Vicious Web Access>):

    ```bash

    #!/bin/bash

    tun2socks --device tun0 --proxy socks5://127.0.0.1:40000 --interface ens4
    sudo ip route add default via <IPv4_gateway_on_tun0> dev tun0 metric <METRIC> table <ROUTE_TABLE>
    # NECESSARY for translation
    sudo ip rule add from <IPv4_network_on_tun0> table <ROUTE_TABLE>
    # With <IPv4_gateway_on_tun0> = <IPv4_inet_on_tun0>
    sudo jool -i nat64 pool4 add --icmp <IPv4_inet_on_tun0> 61001-65535
    sudo jool -i nat64 pool4 add --tcp <IPv4_inet_on_tun0> 61001-65535
    sudo jool -i nat64 pool4 add --udp <IPv4_inet_on_tun0> 61001-65535
    # Verify
    sudo jool -i nat64 pool4 display    # --tcp (default)
    sudo jool -i nat64 pool4 display --udp
    sudo jool -i nat64 pool4 display --icmp

    ```

    And, it worked :rocket: !

  - Don't forget to enable [IP forwarding](https://tao.zz.ac/unix/jool-nat64.html)!

    ```bash

    #!/bin/bash

    # ufw (if)
    sudo sed -i "/net\/ipv4\/ip_forward/s/#//" /etc/ufw/sysctl.conf
    sudo sed -i "/net\/ipv6\/conf\/all\/forwarding/s/#//" /etc/ufw/sysctl.conf
    # Verify (if ufw)
    cat /etc/default/ufw | grep SYSCTL
    cat /etc/ufw/sysctl.conf | grep ip_forward
    cat /etc/ufw/sysctl.conf | grep ipv6/conf/all/forwarding
    # sysctl
    echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
    echo "net.ipv6.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.conf
    sudo sysctl -p
    # Verify
    sudo sysctl -a | grep ip_forward
    sudo sysctl -a | grep ipv6.conf.all.forwarding

    ```

- **Firewall:** With `ufw`, while saving your `ssh`'s ass!

  ```bash

  #!/bin/bash

  # ssh
  sudo ufw allow proto tcp from 0.0.0.0/0 to any port 22
  # nat64
  sudo ufw allow proto tcp from ::/0 to any
  sudo ufw allow proto udp from ::/0 to any
  # Verify
  sudo ufw status
  sudo ufw show added

  ```

  - **Note:** In your VPC Network &rarr; Firewall, add an equivalent set of rules or, FAFO :thumbsup:!

## DNS

A "short" journey from obtaining TLS certificates to implementation of DNS64, coupled with encrypted channels at both the receiving and forwarding ends under identical VPC and VM configurations, if you aren't bored enough!

### Get Certified

A painstaking task of requesting one using a [Public CA](https://docs.cloud.google.com/certificate-manager/docs/public-ca-tutorial) and an [ACME client](https://certbot.eff.org/instructions?ws=apache&os=pip)

- Grant the **Public CA External Account Key Creator** (`roles/publicca.externalAccountKeyCreator`) IAM role
  - [Cloud Console](https://console.cloud.google.com/) &rarr; IAM & Admin &rarr; Service accounts &rarr; Create service account &rarr; Permissions &rarr; Select a role &rarr; External Account Key Creator (Beta)

  - Take note of the **Principal** after account creation

- Enable the [Public CA API](https://docs.cloud.google.com/certificate-manager/docs/reference/public-ca/rest)
  - [Cloud Console](https://console.cloud.google.com/) &rarr; APIs & Services &rarr; Enable APIs and services &rarr; Public Certificate Authority API

- Request an EAB key ID and HMAC
  - [Cloud Console](https://console.cloud.google.com/) &rarr; Activate Cloud Shell (G then S) <svg data-icon-name="devshellIcon" viewBox="0 0 24 24" width="24" height="24" aria-hidden="true"><path d="M19 3H5c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h14c1.1 0 2-.9 2-2V5c0-1.1-.9-2-2-2m0 16H5V5h14zm-6-5-3-3H7l3 3-3 3h3zm4 2v-1h-4v2h4z"></path></svg>

    ```bash

    gcloud publicca external-account-keys create

    ```

- Fire up your VM, install `certbot`, run [apache2](https://ubuntu.com/tutorials/install-and-configure-apache), and then, register an ACME account:

  ```bash

  #!/bin/bash

  sudo certbot register \
    --email "EMAIL_ADDRESS" \
    --no-eff-email \
    --server "SERVER" \
    --eab-kid "EAB_KID" \
    --eab-hmac-key "EAB_HMAC_KEY"

  # Verify
  sudo certbot show_account --server "SERVER"

  ```

  - `EMAIL_ADDRESS`: the **Principal**'s email address
  - `SERVER`: ACME directory URL for the production environment (`https://dv.acme-v02.api.pki.goog/directory`)

- Request certificates

  ```bash

  #!/bin/bash

  sudo certbot certonly \
    --manual \
    --server "SERVER" \
    --domains "DOMAINS"

  # Verify
  sudo ls -lah /etc/letsencrypt/live
  sudo ls -lah /etc/letsencrypt/archive

  ```

- **Automatic renewal:** The _AppArmor_ technique mentioned in [Mastering DNS Privacy](https://medium.com/@kevintim/mastering-dns-privacy-configuring-dns-over-https-doh-and-dns-over-tls-dot-in-bind9-cc6db67aaab3) or [DNS Over HTTPS](https://www.isc.org/blogs/doh-talkdns/) for "recursive read-only access to all files and folders in `/etc/letsencrypt`" didn't work :pensive:! Having recourse to [M9A](https://unix.stackexchange.com/a/757047)'s answer, I had to chalk out a way (ensuring only the latest keys):

  ```bash

  #!/bin/bash

  CERT_DIR="<DOMAINS>"
  #
  CACHE_DIR="/var/cache/bind"
  ##
  SRC_KEYS="$SRC_DIR/archive/$CERT_DIR"
  SRC_LIVE="$SRC_DIR/live/$CERT_DIR"
  #
  CACHE_KEYS="$CACHE_DIR/archive"
  CACHE_LIVE="$CACHE_DIR/live"
  ##
  sudo mkdir -p "$CACHE_KEYS/$CERT_DIR" "$CACHE_LIVE" && \
      sudo chown -R bind:bind "$CACHE_KEYS" && \
      sudo chown -R bind:bind "$CACHE_LIVE"
  CACHE_KEYS="$CACHE_KEYS/$CERT_DIR"
  sudo ls -t "$SRC_KEYS" | head -n 4 | xargs -I {} sudo cp -u "$SRC_KEYS/{}" "$CACHE_KEYS"
  SIZE=$(ls -1A "$CACHE_KEYS" | wc -l)
  [ "$SIZE" -gt 4 ] && \
      ls -t "$CACHE_KEYS" | tail -n 4 | xargs -I {} sudo rm "$CACHE_KEYS/{}"
  sudo cp -ru "$SRC_LIVE" "$CACHE_LIVE"
  sudo chown -R bind:bind "$CACHE_KEYS"
  sudo chown -R bind:bind "$CACHE_LIVE/$CERT_DIR"

  ```

  Finally, a **cron** job! (no, not b\*\*\*job!)

  ```bash

  #!/bin/bash

  echo "0 0,12 * * * root /opt/certbot/bin/python -c 'import random; import time; time.sleep(random.random() * 3600)' && sudo certbot renew -q && bash <path_to_shell_script>" | sudo tee -a /etc/crontab
  # [Monthly] Upgrade certbot
  echo "0 1 1 * * root /opt/certbot/bin/pip install --upgrade --quiet pip certbot certbot-apache" | sudo tee -a /etc/crontab

  ```

### Look Up

- [Configure DNS64](https://oneuptime.com/blog/post/2026-03-20-dns64-bind-ipv6-only-networks/view) with a [recursive resolver](https://oneuptime.com/blog/post/2026-03-20-setup-recursive-dns64-bind/view) whilst using [systemd-resolved alongside BIND](https://oneuptime.com/blog/post/2026-03-04-systemd-resolved-alongside-bind-rhel-9/view), [`allow-query-cache`](https://kb.isc.org/docs/aa-00269), [RRL](https://kb.isc.org/docs/aa-00994), [`window`](https://serverfault.com/questions/490245/bind-dns-rate-limit-and-values-for-responses-per-second-and-window), [`slip`](https://serverfault.com/questions/660992/configuring-rrl-on-a-dns-server-using-bind), limited [`tcp-clients`](https://kb.isc.org/docs/how-does-tcp-clients-work), [`recursive-clients`](https://www.zytrax.com/books/dns/ch7/queries.html), and [DNS over TLS](https://www.crc.id.au/2024/10/06/secure-dns-with-bind-and-dot/) in the `forwarders`:

  ```nginx

  #/etc/bind/named.conf.options

  acl "RFC1918" {
      0.0.0.0/8;
      10.0.0.0/8;
      172.16.0.0/12;
      192.168.0.0/16;
  };

  acl "trusted" {
      any;
  };

  tls cloudflare-tls { remote-hostname "one.one.one.one"; };

  tls google-tls { remote-hostname "dns.google"; };

  tls local-tls {
      key-file "/var/cache/bind/live/<DOMAINS>/privkey.pem";
      cert-file "/var/cache/bind/live/<DOMAINS>/fullchain.pem";
  };

  options {
      directory "/var/cache/bind";

      allow-query { any; };

      listen-on port 53 { <IPv4_address_on_ens4>; };

      listen-on port 853 tls local-tls { any; };

      listen-on port 443 tls local-tls http default { any; };

      listen-on-v6 port 53 { <IPv6_address_on_ens5>; };

      listen-on-v6 port 853 tls local-tls { any; };

      listen-on-v6 port 443 tls local-tls http default { any; };

      recursion yes;

      allow-recursion { trusted; };

      allow-query-cache { trusted; };

      tcp-clients 100;

      recursive-clients 25;

      resolver-query-timeout 5;

      rate-limit {
          responses-per-second 10;
          window 5;
          slip 2;
      };

      dnssec-validation auto;

      forwarders port 853 {
          1.1.1.1 tls cloudflare-tls;
          1.0.0.1 tls cloudflare-tls;

          8.8.8.8 tls google-tls;
          8.8.4.4 tls google-tls;
      };

      forward only;

      dns64 <IPv6_nat64_NSP>/96 {
          clients { any; };

          exclude { <IPv6_nat64_NSP>/96; ::ffff:0:0/96; };

          mapped { !RFC1918; any; };
      };
  };

  ```

- A big :heart: for the public [NAT64 Prefix](https://nat64.xyz/) providers :sparkles:!

- Why `fullchain.pem` and not `cert.pem` in `cert-file`? find out yourself :punch:!

  ```bash

  #!/bin/bash

  openssl s_client -connect <DOMAINS>:853 -showcerts

  ```

:metal: Ciao Adios!
