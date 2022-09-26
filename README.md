# Nftables set fetch

## Description

This script fetches a list of IPs from a URL and creates two sets in nftables.
It has been designed to be used in OpenWrt >= 22.03 using the fw4 firewall.

The two sets are named with the `4` or `6` suffix to it:

    <set-name>4  # IPv4 addresses
    <set-name>6  # IPv6 addresses

The IP list downloaded from the URL is stored and not downloaded again unless it reaches
the stale period which is 3 days by default.

The nftables sets are also not updated unless they are empty or the IP list is refreshed.

## Requirements

- Nftables firewall. The sets are created in the `inet fw4` table.
- The `wget` command.
- The `logger` command.

## Usage

The normal usage is:

    /usr/bin/nftables-set-fetch.sh <set-name> <URL>

To force the update of the IP list and sets, set the variable `FORCE_UPDATE=1`:

    FORCE_UPDATE=1 /usr/bin/nftables-set-fetch.sh <set-name> <URL>

## Example

In the following example I'm downloading a list of DNS resolvers in OpenWrt and
using it to block DNS over HTTPS on the IoT network:

- Add a cron job with to check the list every 10 minutes:

```
# crontab -e
*/10 * * * * /bin/nice -n 19 /usr/bin/nftables-set-fetch.sh resolvers https://public-dns.info/nameservers-all.txt
```

- Add custom nftables rules to `/etc/nftables.d/01-dns-resolvers.nft`. Note that we have to define empty named sets
here to be able to reference them in the rules.

```
set resolvers4 {
    type ipv4_addr
    comment "Set resolvers - IPv4"
}

set resolvers6 {
    type ipv6_addr
    comment "Set resolvers - IPv6"
}

chain user_pre_forward {
    type filter hook forward priority -1; policy accept;
    iifname $iot_devices jump user_pre_forward_iot comment "User-defined rules for pre-forward IoT"
}

chain user_pre_forward_iot {
    ip daddr @resolvers4 tcp dport 443 log prefix "reject DoH: " jump handle_reject
    ip daddr @resolvers4 udp dport 443 log prefix "reject DoH: " jump handle_reject
    ip6 daddr @resolvers6 tcp dport 443 log prefix "reject DoH: " jump handle_reject
    ip6 daddr @resolvers6 udp dport 443 log prefix "reject DoH: " jump handle_reject
}
```
