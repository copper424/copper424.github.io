---
date:
  created: 2024-04-27
  updated: 2024-07-27
categories:
  - Router
tags:
  - Computer Network
---
# OpenWRT IPv6 configuration for Xiaomi router 3 pro

## Configure ULA

In the file located at `/etc/config/network`, we should check that option ula_prefix exists. Otherwise, we should generate a ULA prefix according to RFC 4193.

I use `https://unique-local-ipv6.com/` to generate a valid ULA prefix.

```
config globals 'globals'
        option packet_steering '1'
        option ula_prefix 'fdda:59a8:b467::/48'
```

## Configure DHCPv6
- Luci -> Network -> Interface-> lan -> Edit -> Advanced Settings

  set IPv6 prefix filter to "local (Local ULA)"
 
- Luci -> Network -> Interface-> lan -> Edit -> DHCP server -> IPv6 Settings

  set RA-Service (Route Advertisement Service) to "server mode"
  set DHCPv6-Service to "server mode"
  
- Luci -> Network -> Interface-> lan -> Edit -> DHCP server -> IPv6 RA settings

  set Default router to "forced"
  disable SLAAC i.e uncheck "Enable SLAAC"
  select both M (managed config) and O (other config) in RA flags option
  
## Configure WAN6

- Luci -> Network -> Interface-> wan6 -> Edit -> Advanced Settings

uncheck IPv6 source routing option

## Configure firewall for NAT66 
- Luci -> Network -> Firewall -> Zones -> [WAN => REJECT] -> edit-> Advanced settings

check IPv6 masquerading option
## Restart Router Network

Network and Firewall service are required to restart in order to make changes take effect.

```
service network restart
service firewall restart
```

Or simply reboot the router via `reboot`

# Change the prefixpolicy on Windows
 
 We use `netsh int ipv6 show prefixpolicies` to query the prefix policies. Below block shows an example output. Windows would send network requests via different prefix according to its precedence. Prefix with larger precedence will be tried first.
 
```
Precedence  Prefix         
----------  -------------
    50  ::1/128        Localhost
    40  ::/0           Native IPv6
    35  ::ffff:0:0/96  IPv4
    30  2002::/16      6to4
        5  2001::/32      Teredo
        3  fc00::/7       ULAs
        1  fec0::/10      site-local
        1  3ffe::/16      6bone
        1  ::/96          IPv4compat      
```
 
 Use command `netsh int ipv6 set prefixpolicy fc00::/7 37 13` with administartor permission to overwrite default policy. Then, the policy should be set as shown below.
 
```
Precedence  Prefix         
----------  -------------
    50  ::1/128        
    40  ::/0           Native IPv6
    37  fc00::/7       ULAs <---------- from 3 up to 37
    35  ::ffff:0:0/96  IPv4
    30  2002::/16      
        5  2001::/32      
        1  fec0::/10      site-local
        1  3ffe::/16      
        1  ::/96   
```