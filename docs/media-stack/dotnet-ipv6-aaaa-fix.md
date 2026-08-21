# .NET *arr containers fail external connections on IPv4-only Docker networks (IPv6/AAAA)

## Context

The media stack ran Prowlarr, Sonarr, Radarr, and Lidarr on IPv4-only Docker bridge
networks (`172.20.x.0/24`, no `enable_ipv6`), with IPv6 also disabled network-wide at
the OPNsense layer. Sonarr/Radarr/Lidarr surfaced errors that read like indexer
rate-limiting — `API Request Limit reached for <indexer> (Prowlarr)` — so the obvious
(wrong) conclusion was that the Usenet indexer or Prowlarr was throttling requests. It
was not a rate-limit at all; it was a networking failure being relabelled twice on the
way up the stack.

## Finding

The .NET HTTP stack resolves and tries **IPv6/AAAA first** on dual-stack external hosts
(most Usenet indexers publish both A and AAAA records). The containers have **no IPv6
route** — IPv4-only bridge network, IPv6 off at OPNsense — so the AAAA connection
attempts fail before ever trying the reachable IPv4 address. In the Prowlarr/*arr logs
this shows up as .NET/libc socket errors, not application errors:

- `SocketException (11) Resource temporarily unavailable`
- `Name does not resolve`
- `No data available`

These strings originate in the .NET runtime / libc (the socket layer), not in the *arr
application code — the apps only report what the failed connection threw.

**The double relabelling that hides the root cause:**

1. Prowlarr repeatedly fails to connect to the indexer, records failures, and applies
   its connection-failure backoff — temporarily disabling the indexer. Prowlarr then
   returns HTTP 429 on its own API to the downstream apps during that backoff window.
2. Sonarr/Radarr/Lidarr catch that 429 (`TooManyRequestsException`) and log it at `Warn`
   as `API Request Limit reached for {0}. Disabled for {1}`, where `{0}` renders to the
   synced indexer's name. A networking failure now looks like a provider rate-limit.

**The diagnostic tell — self-inflicted vs. genuine rate-limit:**

- **Self-inflicted (this bug)**: logged by the *arr app with a leading `API ` prefix, and
  the indexer name carries the `(Prowlarr)` marker because it is a Prowlarr-synced
  indexer. Reads:
  `API Request Limit reached for <IndexerName> (Prowlarr). Disabled for <time>`
  Source: `Sonarr/Radarr/Lidarr` `src/NzbDrone.Core/Indexers/HttpIndexerBase.cs`
  (`_logger.Warn("API Request Limit reached for {0}. Disabled for {1}", this, retryTime)`).
  The `(Prowlarr)` text comes from Prowlarr's sync code, which sets
  `Name = $"{indexer.Name} (Prowlarr)"` on each synced indexer
  (`Prowlarr/src/NzbDrone.Core/Applications/{Sonarr,Radarr,Lidarr}/*.cs`); that name is
  what the `{0}` (`this` → `Definition.Name`) renders to.
- **Genuine provider rate-limit**: logged by Prowlarr itself at `Warn` as
  `Request Limit reached for {0}. Disabled for {1}` — **no `API ` prefix**, and the
  indexer name has **no `(Prowlarr)` marker** (Prowlarr talks to its own indexer
  definition directly, not a synced copy). Source:
  `Prowlarr/src/NzbDrone.Core/Indexers/HttpIndexerBase.cs`.

So: `(Prowlarr)` in the indexer name (and the `API ` prefix) means the failure is the
self-inflicted 429 originating inside your own stack; a genuine indexer 429 has neither.

**The fix:**

- **.NET apps (Prowlarr, Sonarr, Radarr, Lidarr)**: set `DOTNET_SYSTEM_NET_DISABLEIPV6=1`.
  This .NET runtime switch makes the apps use only IPv4 addresses and stop attempting
  IPv6 socket connections, so they never try the unreachable AAAA path — which fixes the
  connection-failure backoff those failed connects drive. Scope matters on .NET 6/8: the
  AAAA (IPv6) DNS query is still emitted on the wire; .NET just discards the IPv6 results
  after resolving. True IPv4-only query-narrowing only arrived in ~.NET 10. So a residual
  `Name does not resolve` after this fix points to a DNS-layer problem (Docker's embedded
  resolver at 127.0.0.11 forwarding AAAA to OPNsense Unbound and getting a hard failure),
  not an IPv6-routing problem this switch addresses.
- **Bazarr (Python)**: the `DOTNET_*` switch is inert, so IPv6 is disabled at the
  container level instead via sysctls `net.ipv6.conf.all.disable_ipv6=1` and
  `net.ipv6.conf.default.disable_ipv6=1`.

Prowlarr's own log guidance points straight at this: on connection failure it logs
`Unable to connect to indexer [...]. This is typically caused by DNS/SSL issues. Check
DNS settings, ensure IPv6 is working or disabled, and consider using different DNS
servers or a VPN.` (`Prowlarr/src/NzbDrone.Core/Indexers/HttpIndexerBase.cs`). "IPv6
working or disabled" is the operative phrase — on an IPv4-only network the correct choice
is disabled.

## Source

Discovered via server-log analysis (Prowlarr, Sonarr, Radarr, Lidarr container logs)
cross-referenced against the vendored *arr source in this knowledge base
(`repos/Prowlarr`, `repos/Sonarr`, `repos/Radarr`, `repos/Lidarr`, `repos/bazarr`):

- Self-inflicted 429 log template + genuine rate-limit template (distinct wording):
  `NzbDrone.Core/Indexers/HttpIndexerBase.cs` in each of Sonarr/Radarr/Lidarr (with `API `
  prefix) vs. Prowlarr (without).
- `(Prowlarr)` indexer-name suffix: `Name = $"{indexer.Name} (Prowlarr)"` in
  `Prowlarr/src/NzbDrone.Core/Applications/{Sonarr,Radarr,Lidarr}/*.cs`;
  `IndexerBase.ToString()` returns `Definition.Name`.
- Prowlarr's DNS/SSL/IPv6 troubleshooting log line and wiki reference:
  `Prowlarr/src/NzbDrone.Core/Indexers/HttpIndexerBase.cs`;
  https://wiki.servarr.com/prowlarr/troubleshooting#dns-ssl-connection-issues
- `DOTNET_SYSTEM_NET_DISABLEIPV6` is a documented .NET runtime configuration knob for
  disabling IPv6 in the socket/name-resolution layer.
