# Architecture

## iOS: from Flutter to the tunnel

```mermaid
flowchart TB
  subgraph App["Runner (app process)"]
    Dart["Flutter / Dart<br/>connection controller"]
    Bridge["Swift plugin<br/>MethodChannel velaris/ios_tunnel"]
    Mgr["NETunnelProviderManager<br/>VPN profile + connect-on-demand"]
    Dart -- "prepare / start / stop / hold / health / renew" --> Bridge
    Bridge --> Mgr
  end
  subgraph Ext["Packet Tunnel extension (separate process)"]
    PTP["NEPacketTunnelProvider (Swift)"]
    Xray["Xray-core<br/>VLESS + REALITY"]
    T2S["tun2socks"]
    HB["Lease heartbeat<br/>+ self-recovery"]
    PTP --> T2S --> Xray
    PTP --> HB
  end
  Mgr -- "starts / stops" --> PTP
  Bridge <-- "sendProviderMessage / handleAppMessage" --> PTP
  App <-. "App Group shared state" .-> Ext
  Xray -- "encrypted tunnel" --> Server["Entry server"]
  HB -- "HTTPS heartbeat, lease + manifest renewal" --> API["Control plane API"]
```

Who does what:

- **Dart:** sign-in, choosing a location, getting the lease and manifest, verifying the
  manifest signature, and the UI state machine.
- **Runner plugin:** owns the single VPN profile, the on-demand rules, the session and expiry
  stored in the profile, and health and status queries.
- **Extension:** applies the network settings (routes, DNS, IPv6), runs Xray, enforces the
  session locally, sends heartbeats while the app is suspended, holds traffic on fatal errors,
  and recovers on its own when the app did not ask for the hold.

## Control plane and data plane

```mermaid
flowchart LR
  App["Velaris app<br/>(iOS, Android, Windows)"]
  subgraph CP["Control plane"]
    API["FastAPI<br/>auth, devices, leases,<br/>manifests, telemetry"]
    PG[("PostgreSQL<br/>users, entitlements, devices,<br/>leases, attempts, usage")]
    Keys["Ed25519 manifest keys<br/>+ keyset renewal timer"]
    API --> PG
    Keys --> API
  end
  subgraph DP["Data plane"]
    SA["session-authority<br/>patched Xray-core"]
  end
  subgraph Ops["Operations"]
    P["Prometheus"] --> AM["Alertmanager"] --> N["Push notifications"]
    BK["Daily DB backup<br/>+ restore test"]
  end
  App -- "HTTPS" --> API
  API -- "issue / revoke<br/>session credential" --> SA
  App -- "VLESS + REALITY" --> SA
  SA --> Net((Internet))
  P -. scrape .-> API
  P -. scrape .-> PG
  P -. scrape .-> SA
  BK -.-> PG
```

## Target topology (simulated only)

```mermaid
flowchart LR
  C["Client"] --> SA1["Shield A<br/>(HAProxy)"]
  C --> SB["Shield B<br/>(HAProxy)"]
  SA1 -- "WireGuard" --> Core["Core<br/>Xray REALITY"]
  SB -- "WireGuard" --> Core
  Core --> I((Internet))
```

The Shields are replaceable edge nodes, and the Core is reachable only through
WireGuard. This topology runs in containers for testing, including failover
between Shields. It has not been deployed on separate servers yet.
