# OPNsense Privacy DNS: DoT vs DoH vs Full Recursion

## Context

Deciding how to make OPNsense's Unbound resolver private: forward to a privacy
provider over DoT/DoH, or stop forwarding and recurse directly. Covers the exact
GUI paths, config values, provider endpoints, and the privacy tradeoffs — all
verified against the OPNsense source vendored in this KB
(`repos/opnsense-core`, `repos/opnsense-plugins`), not from memory.

## The core tradeoff

"Privacy DNS via DoT/DoH to a public resolver" only swaps *who* observes you:
instead of the ISP seeing plaintext queries, one company (Cloudflare/Quad9/etc.)
sees 100% of them. A single party still sees the entire query stream.

**Full recursion eliminates that single observer.** Unbound talks directly to
root/TLD/authoritative servers; each authoritative server sees only the fragment
it is authoritative for. Nobody sees the whole picture. It is native (no plugin),
DNSSEC-validated end-to-end, and cold-cache latency is negligible on a warm
homelab cache with local overrides.

| Option | Privacy | Effort | Who sees your queries |
|--------|---------|--------|----------------------|
| Full recursion (no upstream) | Best | One toggle, native | Nobody sees the whole stream — each authoritative server sees only its fragment |
| DoT to a no-log provider | Medium | Native, easy | That one provider sees everything (encrypted from the ISP) |
| DoH/DNSCrypt via dnscrypt-proxy, multi-server + relays | Strong | Extra plugin | Trust split across resolvers + relays; blends with HTTPS |

**Recommendation for this homelab: full recursion with DNSSEC + QNAME
minimisation.** Only layer encryption-to-upstream if hiding DNS from the ISP on
the wire is a hard requirement — and then prefer dnscrypt-proxy with
load-balanced servers + anonymized relays over pinning everything to one DoT
provider. Single-provider DoT is the easy-but-one-company-sees-all middle ground.

## Finding: Full recursion (recommended)

1. `Services → Unbound DNS → Query Forwarding` → disable forwarding and remove
   all upstream servers (e.g. `8.8.8.8` / `8.8.4.4`). Add no DoT entries.
2. `Services → Unbound DNS → General` → confirm **Enable DNSSEC Support** is on.
3. Leave **Strict QNAME Minimisation** *off* (Advanced tab) — strict breaks many
   domains. Base QNAME minimisation is already active (see defaults below).

Unbound now recurses directly. No third party sees the full query stream.
Downside: the WAN IP is visible to authoritative servers (fragmented, never
aggregated).

## Finding: DoT (DNS-over-TLS) — native in Unbound

`Services → Unbound DNS → DNS over TLS` tab → `+`. Fields (source:
`forms/dialogDot.xml`):

- **Enabled**: on
- **Domain**: leave **blank** to forward ALL queries (catch-all). A value scopes
  forwarding to that one domain.
- **Server IP**: upstream resolver IP
- **Server Port**: `853`
- **Verify CN**: the upstream's TLS cert common name. **Always set this** —
  blank accepts self-signed certs and is MITM-vulnerable.

A blank-Domain DoT entry emits `forward-zone: name: "."` (source:
`templates/.../Unbound/core/dot.conf`), putting Unbound into full forwarding
mode — it stops recursing and hands the entire query stream to that upstream.
This is distinct from the *Query Forwarding* tab's master toggle, which forwards
**plaintext** to system/ISP nameservers (do not use it for privacy).

DNSSEC validation still happens locally when forwarding, provided the upstream
returns RRSIGs (Quad9/Cloudflare/Mullvad/AdGuard all qualify).

### Provider DoT endpoints (`IP:853#Verify-CN`)

| Provider | DoT (primary / secondary) | Verify CN | DoH URL | Note |
|----------|---------------------------|-----------|---------|------|
| Quad9 (secured, malware+DNSSEC) | `9.9.9.9` / `149.112.112.112` | `dns.quad9.net` | `https://dns.quad9.net/dns-query` | Switzerland, no-log |
| Quad9 (unfiltered, no DNSSEC) | `9.9.9.10` / `149.112.112.10` | `dns10.quad9.net` | `https://dns10.quad9.net/dns-query` | |
| Cloudflare | `1.1.1.1` / `1.0.0.1` | `cloudflare-dns.com` | `https://cloudflare-dns.com/dns-query` | US; ~24h logs |
| Cloudflare malware / family | `1.1.1.2`·`1.0.0.2` / `1.1.1.3`·`1.0.0.3` | `security.` / `family.cloudflare-dns.com` | matching subdomain | |
| Mullvad (no filter) | `194.242.2.2` | `dns.mullvad.net` | `https://dns.mullvad.net/dns-query` | Sweden, no-log, no account |
| Mullvad adblock / base / all | `194.242.2.3` / `.4` / `.9` | `adblock.` / `base.` / `all.dns.mullvad.net` | matching subdomain | tiers cumulative |
| AdGuard (ads+trackers) | `94.140.14.14` / `94.140.15.15` | `dns.adguard-dns.com` | `https://dns.adguard-dns.com/dns-query` | Cyprus; anonymized stats |
| AdGuard unfiltered / family | `94.140.14.140`·`.141` / `94.140.14.15`·`94.140.15.16` | `unfiltered.` / `family.adguard-dns.com` | matching subdomain | |

For a no-log jurisdiction pick Mullvad or Quad9 over Cloudflare (US). **dns0.eu
shut down ~Oct 2025 — do not configure it.** EU non-profit replacement: DNS4EU
(joindns4.eu).

## Finding: DoH (DNS-over-HTTPS) — needs the dnscrypt-proxy plugin

Unbound cannot forward over DoH. OPNsense's supported path:

- Plugin: `os-dnscrypt-proxy` (source: `Makefile` → `PLUGIN_DEPENDS=dnscrypt-proxy2`).
  Install via `System → Firmware → Plugins`.
- Config: `Services → DNSCrypt-Proxy → Configuration`.
- Architecture: `Unbound (:53) → dnscrypt-proxy (:5353) → DoH/DNSCrypt/ODoH upstreams`.
  dnscrypt-proxy defaults to listening on `0.0.0.0:5353` (source: `General.xml`)
  so Unbound keeps `:53`. Wire Unbound to it via *Query Forwarding* → forward to
  `127.0.0.1` port `5353`.
- Relevant defaults (source `General.xml`): `doh_servers=true`,
  `dnscrypt_servers=true`, `odoh_servers=false`, `require_nolog=true`,
  `require_dnssec=false`, plus a **Relay List** for anonymized DNSCrypt (splits
  "who you are" from "what you ask"). Strongest forwarding privacy posture:
  multiple load-balanced servers + anonymized relays.

## Finding: DNSSEC + QNAME minimisation defaults

- **DNSSEC**: ON by default on fresh installs (source: `InitialSetup.php` sets
  `unbound.dnssec` default `true`). Writes `module-config: "validator iterator"`
  + `auto-trust-anchor-file: /var/unbound/root.key` (source: `unbound.inc`).
- **QNAME minimisation**: enabled implicitly. OPNsense does not write the base
  `qname-minimisation` directive (absent in source), relying on Unbound's
  compiled-in default `yes` (default since Unbound 1.7.2, June 2018). The GUI
  only exposes **Strict QNAME Minimisation** (Advanced tab), OFF by default
  (source: model `qnameminstrict` has no default) — leave it off; strict breaks
  many domains.
- **Also on by default** (source: `Unbound.xml`): `aggressive-nsec` (RFC 8198)
  and `harden-below-nxdomain`, both `1`. Good for privacy; no action needed.
- Caveat: QNAME minimisation gives no privacy benefit against the forwarder
  itself — a single DoT/DoH upstream sees the full name anyway. It only helps on
  the recursion path.

## OPNsense-side, not Ansible-managed

All of the above is `Services → Unbound DNS …` on OPNsense, which is not
Ansible-managed in IoTBase (the API user is read-only). Any choice here is a
manual GUI change. No IoTBase repo change is required unless new `.home` host
overrides are added (which would also need mirroring into
`headscale_dns_extra_records`).

## Source

- OPNsense source vendored in this KB: `repos/opnsense-core`,
  `repos/opnsense-plugins`
- https://docs.opnsense.org/manual/unbound.html
- https://unbound.docs.nlnetlabs.nl/en/latest/manpages/unbound.conf.html
- Provider docs: quad9.net, developers.cloudflare.com/1.1.1.1, mullvad.net,
  adguard-dns.io
