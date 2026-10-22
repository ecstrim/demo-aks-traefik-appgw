# Application Gateway wiring

Application Gateway (v2) terminates TLS and forwards plain HTTP to the Traefik
internal load balancer. Traefik routes by `Host` header to the right namespace.

```
Client --HTTPS--> Application Gateway --HTTP:80--> Traefik ILB --> Traefik pods --> echo-a / echo-b
```

These settings are not applied by this repo. Create them in your own
Terraform, Bicep or portal. They have not been tested against a live gateway
as part of this demo, so check them against the Application Gateway docs.

## Backend pool

| Setting | Value |
|---------|-------|
| Target type | IP address |
| Target | The `loadBalancerIP` from `traefik/values.yaml` |

## Backend settings (HTTP settings)

| Setting | Value | Why |
|---------|-------|-----|
| Protocol / port | HTTP / 80 | TLS ends at the gateway |
| Override with new host name | No (use the incoming `Host`) | The IngressRoutes match on `Host(...)`. If the gateway rewrites the host, routing breaks |
| Custom probe | Yes | See below |
| Request timeout | 60 s or higher | Default is 20 s |

## Health probe

| Setting | Value |
|---------|-------|
| Protocol | HTTP |
| Path | `/ping` |
| Host | Any value, or pick from backend settings |
| Expected status | 200-399 |

`/ping` is served by Traefik on the `web` entrypoint (`--ping.entryPoint=web`)
and does not depend on any IngressRoute, so it stays healthy while you change
routes.

## Listener and rule

- One HTTPS listener per hostname (or a multi-site listener with a wildcard
  certificate), with the certificate in Key Vault.
- A routing rule from the listener to the backend pool and settings above.
- Optional: an HTTP listener with a redirect rule to HTTPS.

## Network

- The Application Gateway subnet must be able to reach the Traefik ILB IP on
  port 80. Check NSGs on the AKS subnet.
- Put the Application Gateway subnet CIDR in
  `ports.web.forwardedHeaders.trustedIPs` in `traefik/values.yaml`. Traefik
  then trusts the `X-Forwarded-*` headers the gateway sends, and nobody else.
- The Traefik ILB must live in a subnet the gateway can route to (same VNet or
  a peered one).
