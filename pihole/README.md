# Dedicated Pi-hole DNS and DHCP Server

A dedicated Pi-hole appliance running directly on Ubuntu Server.

## Purpose

Pi-hole was moved from the primary application server to a separate physical machine so that DNS and DHCP could operate independently of Docker, Traefik, and application-server maintenance.

This design provides:

- Independent DNS availability
- Native DHCP support
- No Docker host-networking complications
- No web-port conflicts with Traefik
- Easier troubleshooting
- Continued network services while the main server is rebooted

## Architecture

```text
Internet
   |
ISP Gateway
   |
LAN Switch
   |
   +-- Dedicated Pi-hole appliance
   |     +-- DNS filtering
   |     +-- DHCP
   |     +-- Static DHCP reservations
   |     +-- Tailscale administration
   |
   +-- Main Ubuntu application server
   |     +-- Docker
   |     +-- Traefik
   |     +-- Home Assistant
   |     +-- Plex
   |     +-- Samba
   |
   +-- Windows workstation
   +-- Phones, tablets, TVs, and other clients
```

## Network Responsibilities

### ISP Gateway

The ISP gateway continues to provide:

- Internet routing
- NAT
- Firewall services
- Wireless access
- Default gateway service

Its IPv4 DHCP server is disabled.

### Pi-hole Appliance

The dedicated Pi-hole appliance provides:

- IPv4 DHCP leases
- Network-wide DNS filtering
- Static DHCP reservations
- Local DNS visibility
- Upstream DNS forwarding
- DNSSEC validation
- Secure remote administration through Tailscale

The appliance uses a host-level static RFC1918 address configured through Netplan.

## DHCP

Pi-hole is the only active IPv4 DHCP server on the LAN.

It advertises:

- The ISP gateway as the default router
- Pi-hole itself as the DNS server
- A one-day lease duration

Important infrastructure devices receive consistent addresses through MAC-based DHCP reservations.

## DNS

DHCP clients are instructed to use Pi-hole as their DNS resolver.

Pi-hole forwards permitted requests to Cloudflare's public DNS resolvers:

```text
1.1.1.1
1.0.0.1
```

DNSSEC is enabled to validate signed DNS responses.

## IPv6

IPv6 is temporarily disabled on the ISP gateway.

The gateway was advertising its own IPv6 DNS resolver to clients, which allowed DNS requests to bypass Pi-hole despite correct IPv4 DHCP configuration.

IPv6 will be restored after deploying routing equipment that provides control over:

- Router advertisements
- RDNSS
- DHCPv6
- Prefix delegation
- Advertised IPv6 DNS servers
- Firewall policy
- VLAN routing

## Security

- Pi-hole runs on a dedicated Linux host
- SSH key authentication is used for administration
- Management services are not exposed directly to the public internet
- Tailscale provides secure remote administrative access
- Only one DHCP server is active on the LAN
- Internal addresses and MAC addresses are excluded from public documentation

## Useful Checks

Check interface addressing:

```bash
ip -4 -br addr
```

Check routing:

```bash
ip route
```

Check Pi-hole FTL:

```bash
systemctl status pihole-FTL
```

Check whether DHCP is listening:

```bash
sudo ss -lunp | grep ':67'
```

Test DNS resolution:

```bash
dig example.com
```

The responding DNS server should be the dedicated Pi-hole appliance.

## Troubleshooting Lessons

This migration reinforced several useful troubleshooting principles:

1. Identify which subsystem owns the problem.
2. Compare expected behavior with observed behavior.
3. Verify whether clients use DHCP or manual network settings.
4. Confirm which DHCP server issued each lease.
5. Distinguish the default gateway from the DNS server.
6. Remember that IPv6 can distribute DNS independently of IPv4 DHCP.
7. Validate changes from the client side instead of trusting configuration pages alone.

## Future Work

- Export and back up the Pi-hole configuration
- Add useful local DNS records
- Remove the legacy Pi-hole instance
- Expand the DHCP pool if required
- Add monitoring and alerting
- Replace the ISP gateway's routing functions
- Introduce VLANs for servers, IoT devices, and cameras
- Restore IPv6 with Pi-hole as the advertised DNS resolver
