# demo-aks-traefik-appgw

A small, working-shape example of Traefik v3 on AKS behind Azure Application
Gateway. Two namespaces, each with an echo server, routed by hostname.

```
Client --HTTPS--> Application Gateway --HTTP--> Traefik (internal LB)
                                                   |-- Host(echo-a.example.com) --> ns echo-a / echo
                                                   '-- Host(echo-b.example.com) --> ns echo-b / echo
```

> **Demo only.** This is an example, not a supported product or a hardened
> production setup. No warranty, no support. If you use it, you are on your
> own: review it, test it, and adapt it to your environment first. See
> [LICENSE](LICENSE) (MIT).

## Layout

| Path | What |
|------|------|
| `traefik/values.yaml` | Helm values for the Traefik chart |
| `traefik/pdb.yaml` | PodDisruptionBudget |
| `traefik/servicemonitor.yaml` | Optional Prometheus scrape config |
| `apps/echo-a`, `apps/echo-b` | Namespace, Deployment, Service, Middleware, IngressRoute |
| `docs/appgateway.md` | How to point Application Gateway at Traefik |

## Design notes

- **TLS ends at Application Gateway.** Traefik only serves HTTP on the `web`
  entrypoint. No certificates live in the cluster.
- **Internal load balancer.** Traefik is not on the internet. Only the gateway
  reaches it.
- **Namespace isolation.** `allowCrossNamespace: false` means an IngressRoute
  can only use services and middlewares from its own namespace. Each demo app
  has its own rate-limit middleware (`echo-a`: 10 req/s, `echo-b`: 2 req/s).
- **Trusted proxies.** Only the gateway subnet is trusted for `X-Forwarded-*`.
- **Metrics** are on a separate ClusterIP service, never on the load balancer.

## Before you start

Edit `traefik/values.yaml` and replace the three `CHANGE ME` values:
the gateway subnet CIDR, the AKS subnet name, and a free IP in that subnet.

## Install

Check you are on the right cluster first: `kubectl config current-context`.

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update

helm upgrade --install traefik traefik/traefik \
  --version 42.1.3 \
  --namespace traefik --create-namespace \
  -f traefik/values.yaml

kubectl apply -f traefik/pdb.yaml
kubectl apply -k apps/
```

Optional, if you run the Prometheus Operator: `kubectl apply -f traefik/servicemonitor.yaml`.

## Try it

Point the gateway at the Traefik IP (see `docs/appgateway.md`), then:

```bash
curl -s https://echo-a.example.com/ | jq '.environment.DEMO_NAMESPACE'   # "echo-a"
curl -s https://echo-b.example.com/ | jq '.environment.DEMO_NAMESPACE'   # "echo-b"
```

Without a gateway, from inside the cluster:

```bash
kubectl -n traefik port-forward deploy/traefik 8000:8000
curl -s -H 'Host: echo-a.example.com' http://localhost:8000/
curl -s -H 'Host: echo-b.example.com' http://localhost:8000/
```

Hit `echo-b` quickly in a loop and you will see `429 Too Many Requests`
once its rate limit (2 req/s, burst 4) is exceeded. `echo-a` stays fine.

## Validate without a cluster

```bash
helm template traefik traefik/traefik --version 42.1.3 -n traefik -f traefik/values.yaml
kubectl kustomize apps/
```

## Gotchas

- Keep the `match:` rule on one line. A newline silently drops the router.
- The probe port in the chart (`deployment.healthchecksPort: 8000`) must match
  the entrypoint that serves `/ping`.
- Chart 42.x renamed `logs` to `log` and `accessLog`. Older value files will
  fail schema validation.

## License

MIT, provided as is and without warranty. See [LICENSE](LICENSE).
