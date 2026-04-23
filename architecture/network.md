# Network architecture

TODO. Describe:

- Physical layout (router, switches, VLANs at a conceptual level).
- Logical segmentation (management, services, storage, IoT, guest).
- Edge: Cloudflare Tunnel for the public showcase, Nginx Proxy Manager for
  internal reverse proxying.
- Kubernetes ingress path from client to pod.
- Observability traffic (Prometheus scrape paths, Loki push path).

Avoid publishing real IP ranges. Diagrams should use generic placeholders.
