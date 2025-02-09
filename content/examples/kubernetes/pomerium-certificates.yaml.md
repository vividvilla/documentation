```yaml title="pomerium-console-certificate.yaml"
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: pomerium-cert
  namespace: pomerium
spec:
  secretName: pomerium-tls
  issuerRef:
    name: pomerium-issuer
    kind: Issuer
  usages:
    - server auth
    - client auth
  dnsNames:
    - pomerium-proxy.pomerium.svc.cluster.local
    - pomerium-authorize.pomerium.svc.cluster.local
    - pomerium-databroker.pomerium.svc.cluster.local
    - pomerium-authenticate.pomerium.svc.cluster.local
    - authenticate.localhost.pomerium.io
    # TODO - If you're not using the Pomerium Ingress controller, you may want a wildcard entry as well.
    #- "*.localhost.pomerium.io" # Quotes are required to escape the wildcard
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: pomerium-redis-cert
  namespace: pomerium
spec:
  secretName: pomerium-redis-tls
  issuerRef:
    name: pomerium-issuer
    kind: Issuer
  dnsNames:
    - pomerium-redis-master.pomerium.svc.cluster.local
    - pomerium-redis-headless.pomerium.svc.cluster.local
    - pomerium-redis-replicas.pomerium.svc.cluster.local
```
