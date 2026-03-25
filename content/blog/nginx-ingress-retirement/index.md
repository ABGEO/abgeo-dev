+++
title = "Nginx Ingress Is Dead: Your Complete Guide to What Comes Next"
date = "2026-03-25T19:54:48+04:00"
cover = "/blog/nginx-ingress-retirement/images/cover.png"
images = [
    "/blog/nginx-ingress-retirement/images/cover.png"
]
tags = ["Kubernetes", "DevOps", "Gateway API", "Ingress"]
keywords = [
    "nginx ingress controller retirement",
    "kubernetes nginx ingress end of life",
    "nginx ingress replacement",
    "kubernetes gateway api migration",
    "ingress controller alternatives",
    "migrate from nginx ingress",
    "envoy gateway kubernetes",
    "traefik ingress controller",
    "cilium gateway api",
    "kubernetes ingress controller 2026",
    "gateway api vs ingress",
    "nginx ingress eol",
    "kubernetes traffic management",
    "helm chart gateway api migration"
]
description = "The Kubernetes team has retired the nginx ingress controller. The repo is archived, no more releases, no security patches. Here's your complete guide to the three migration paths: Gateway API, dual-support controllers, and alternative ingress controllers, with real examples and honest comparisons."
showFullContent = false
readingTime = true
+++

For many years, the nginx ingress controller was the preferable choice when running a Kubernetes cluster, whether it was a simple few-node homelab setup or a production-grade deployment serving millions of requests. You installed it, wrote a couple of Ingress manifests, and moved on with your life. It just worked.

That era is over. The Kubernetes team [announced the retirement](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/) of the [ingress-nginx](https://github.com/kubernetes/ingress-nginx/) project back in November 2025, and as of March 2026 the project is officially retired. The repository is archived and marked read-only. No more releases, no bug fixes, no security patches.

You can still run the old controller versions in your cluster, but they are no longer maintained and will not receive any support or security updates. Running unpatched infrastructure in production is a risk that only grows with time, and I would highly not recommend it.

In this post, we'll go through the different options you have: migrating to the Gateway API, picking a controller that supports both old and new APIs, or simply swapping nginx for another Ingress controller.

## What Happened and Why

The retirement wasn't a sudden decision. The project had been struggling for a while:

- **Maintainer burnout.** Only 1-2 people were maintaining what was arguably one of the most widely deployed Kubernetes components. On their own time, unpaid.
- **Technical debt.** Design decisions that were once considered flexibility features became security liabilities. The "snippets" annotation system, which allowed injecting arbitrary NGINX configuration directives, is a prime example.
- **Failed replacement.** A project called [InGate](https://github.com/kubernetes-sigs/ingate) was started as a successor but never reached maturity and was also retired.
- **Security concerns.** With so few maintainers, the project could no longer guarantee timely security updates.

Despite being one of the most popular Kubernetes projects, the community couldn't sustain it. That's a sobering reminder that open-source infrastructure needs active investment, not just passive consumption.

### What This Means in Practice

- Your **existing deployments will continue to function**. Nothing suddenly broke.
- Helm charts and container images are **still available**. They weren't deleted.
- But there are **no new releases**, no bug fixes, and most importantly, **no security patches**.

Every vulnerability discovered from now on stays unpatched. That's the real risk here.

You can check if you're running ingress-nginx with:

```bash
kubectl get pods --all-namespaces --selector app.kubernetes.io/name=ingress-nginx
```

If that returns results, it's time to move.

## Option 1: Migrate to the Gateway API (Recommended)

I really encourage you to test out the Gateway API if you haven't already. This is also the recommended path from the Kubernetes team, and for good reason.

### What Is the Gateway API?

The [Gateway API](https://gateway-api.sigs.k8s.io/) is the modern successor to the Ingress API. It manages incoming traffic using a different approach. Instead of a single Ingress resource where anything beyond basic routing required controller-specific annotations, the Gateway API introduces a **role-oriented model** with dedicated resources for each concern.

The core resources are:

- **GatewayClass**: Defines a class of Gateways, similar to how IngressClass works. Typically provided by your infrastructure team or cloud provider.
- **Gateway**: Represents the actual load balancer infrastructure. Listeners, ports, TLS configuration. This is where your platform team defines how traffic enters the cluster.
- **HTTPRoute**: Defines routing rules. Path matching, header-based routing, traffic splitting, redirects. This is what application developers work with.

The big difference from Ingress is the **separation of concerns**. With Ingress, a single resource tried to handle everything, and anything beyond basic host/path routing required annotations that weren't portable between controllers. With Gateway API, capabilities like header matching, traffic weighting, and request manipulation are first-class citizens, standardized across all implementations.

Here's what a basic HTTPRoute looks like:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
spec:
  parentRefs:
    - name: my-gateway
  hostnames:
    - "app.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: my-app-service
          port: 80
```

And here's something the Ingress API could never do natively, traffic splitting between two versions of a service:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app-canary
spec:
  parentRefs:
    - name: my-gateway
  hostnames:
    - "app.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: my-app-v1
          port: 80
          weight: 90
        - name: my-app-v2
          port: 80
          weight: 10
```

No annotations, no controller-specific hacks. Just a standard API that works the same everywhere.

### Gateway API Implementations Compared

The Gateway API is a specification, not a product. You need an implementation to actually run it. Here are the ones worth looking at.

#### Envoy Gateway

[Envoy Gateway](https://github.com/envoyproxy/gateway) was built specifically for the Gateway API from the ground up. It's powered by [Envoy Proxy](https://www.envoyproxy.io/), one of the most proven proxies out there, and lives under the CNCF as part of the Envoy umbrella.

**Pros:**
- Designed around Gateway API from day one, not an adapter layer on top of something older
- Excellent spec conformance
- Envoy's proven performance and extensibility (WASM filters, rate limiting, auth)
- Clean separation between control plane and data plane
- Active CNCF backing

**Cons:**
- v1.0 came out in 2024, so it has less production mileage than older tools
- Fewer tutorials and blog posts compared to Traefik or Istio
- Envoy configuration complexity can leak through in edge cases

**Helm chart**: `helm install eg oci://docker.io/envoyproxy/gateway-helm`

#### Cilium

[Cilium](https://github.com/cilium/cilium) is primarily a CNI plugin that uses eBPF for networking, security, and observability. Its Gateway API support is built directly into the networking layer, so you don't need a separate controller deployment.

**Pros:**
- eBPF gives you kernel-level performance that's hard to match
- Unified stack: CNI, service mesh, and gateway all in one
- Built-in observability via [Hubble](https://github.com/cilium/hubble)
- Transparent encryption with WireGuard or IPsec
- CNCF Graduated project, powers GKE Dataplane V2

**Cons:**
- You have to adopt Cilium as your CNI. That's a big infrastructure decision, not just swapping an ingress controller.
- Needs Linux kernel 4.19+ (ideally 5.10+)
- Gateway features are newer and may not cover everything dedicated controllers offer
- If you only need ingress/gateway functionality, it's overkill

#### Traefik

[Traefik](https://github.com/traefik/traefik) is a mature, cloud-native edge router with native Kubernetes integration. It supports Gateway API alongside its own IngressRoute CRD and the standard Ingress API.

**Pros:**
- Huge community, around 52k GitHub stars
- Built-in Let's Encrypt / ACME support
- Automatic service discovery
- Comes with a dashboard for monitoring and debugging
- Default ingress controller in k3s

**Cons:**
- Distributed rate limiting, WAF, and OIDC require Traefik Enterprise (paid)
- Heavy use of the IngressRoute CRD creates vendor lock-in
- Proxy and control plane run as a single binary

#### Istio

[Istio](https://github.com/istio/istio) is a full service mesh that also works as a Gateway API implementation. It handles both north-south (ingress) and east-west (service-to-service) traffic.

**Pros:**
- Full service mesh: mTLS, traffic management, observability
- Excellent Gateway API conformance
- Supports the GAMMA initiative for mesh traffic via Gateway API
- Massive community and enterprise adoption

**Cons:**
- It's a service mesh, not just an ingress controller. The operational complexity reflects that.
- Higher resource footprint
- Steep learning curve
- If you don't need mesh features, you're paying for a lot of overhead you won't use

#### Kong

[Kong Ingress Controller](https://github.com/Kong/kubernetes-ingress-controller) bridges API gateway features with Kubernetes-native traffic management. Under the hood it runs [Kong Gateway](https://github.com/Kong/kong), which is built on NGINX/OpenResty.

**Pros:**
- Full API gateway capabilities: rate limiting, auth, request transformation
- Over 100 plugins
- NGINX-based data plane, so the migration from ingress-nginx feels familiar
- Strong enterprise support option

**Cons:**
- Many of the useful plugins are behind Kong Enterprise (paid)
- Heavier on resources than simpler controllers
- Vendor-specific CRDs (KongPlugin, KongConsumer) add complexity
- For simple routing, it's more than you need

#### Quick Comparison

| Implementation | Conformance | Performance | Complexity | Best For |
|---|---|---|---|---|
| Envoy Gateway | Excellent | Excellent | Medium | Standards-first Gateway API setup |
| Cilium | Good | Excellent (eBPF) | High | Already using or adopting Cilium as CNI |
| Traefik | Good | Good | Low-Medium | General purpose, ease of use |
| Istio | Excellent | Good | High | Full service mesh needs |
| Kong | Good | Good | Medium-High | API gateway features and plugins |

### When Your Charts Don't Support Gateway API Yet

The Gateway API is still in its early adoption stage by the OSS community, and it might not be supported by all the Helm charts and deployments you're using. If that's the case, there are a couple of workarounds.

#### Charts with Raw Manifest Support

Some Helm charts let you inject arbitrary Kubernetes manifests through their values. This is usually exposed as `extraManifests`, `extraObjects`, or `rawResources`. If your chart has this, you can inject an HTTPRoute directly:

```yaml
# values.yaml
extraManifests:
  - apiVersion: gateway.networking.k8s.io/v1
    kind: HTTPRoute
    metadata:
      name: my-app
    spec:
      parentRefs:
        - name: my-gateway
      hostnames:
        - "app.example.com"
      rules:
        - matches:
            - path:
                type: PathPrefix
                value: /
          backendRefs:
            - name: my-app
              port: 80
```

Check your chart's documentation for the exact field name. It varies between charts, but the pattern is the same.

#### Creating a Wrapper Chart

If the chart doesn't support raw manifests at all, nothing prevents you from creating the resources yourself. I recommend creating a small wrapper chart that includes the original chart as a dependency and adds your Gateway API resources on top.

Here's how:

```yaml
# my-app-wrapper/Chart.yaml
apiVersion: v2
name: my-app-wrapper
version: 1.0.0
dependencies:
  - name: my-app
    version: "2.5.0"
    repository: "https://charts.example.com"
```

Then add your Gateway API resources as templates:

```yaml
# my-app-wrapper/templates/httproute.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: {{ .Release.Name }}
spec:
  parentRefs:
    - name: {{ .Values.gateway.name | default "my-gateway" }}
  hostnames:
    - {{ .Values.hostname | quote }}
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: {{ .Release.Name }}-my-app
          port: {{ .Values.service.port | default 80 }}
```

```yaml
# my-app-wrapper/values.yaml
hostname: "app.example.com"
gateway:
  name: "my-gateway"
service:
  port: 80

# Pass-through values to the upstream chart
my-app:
  # ... original chart values go here
```

Build dependencies and install:

```bash
helm dependency build my-app-wrapper/
helm install my-app ./my-app-wrapper -f my-values.yaml
```

This keeps everything version-controlled and your Gateway API resources live right next to the application they belong to.

## Option 2: The Bridge Strategy

Here's something worth considering. Several controllers support **both** the Ingress API and the Gateway API at the same time. You can deploy one of these as a drop-in replacement for nginx ingress, keep your existing Ingress resources working, and gradually convert routes to Gateway API whenever you're ready.

If you have a large number of Ingress resources and don't want to do a big-bang migration, this is the pragmatic choice.

Controllers that support both:

- **Traefik**: Supports Ingress, its own IngressRoute CRD, and Gateway API simultaneously
- **Cilium**: Supports both Ingress and Gateway API through its eBPF-based proxy
- **Kong**: Supports Ingress resources alongside Gateway API HTTPRoute and GRPCRoute
- **HAProxy**: Supports [Ingress](https://github.com/haproxytech/kubernetes-ingress) with [growing Gateway API conformance](https://github.com/haproxytech/kubernetes-ingress)

The migration looks like this:

1. Deploy the new controller alongside nginx ingress
2. Migrate Ingress resources to the new controller (often just changing the `ingressClassName`)
3. Once things are stable, start converting high-value routes to HTTPRoute
4. Decommission nginx ingress
5. Keep migrating remaining routes to Gateway API over time

No rush, no big bang, no downtime. It might make sense to pick one of these even if you don't plan to use Gateway API right away, just so you have the option later without another controller swap.

## Option 3: Stay on the Ingress API

If Gateway API is not an option for you right now, or you want to stick with the old trusted Ingress API, that's a valid choice. The Ingress API itself isn't going anywhere. Here are a couple of controllers that can replace nginx ingress in your environments.

### Traefik

[Traefik](https://github.com/traefik/traefik) is probably the most natural replacement. It's the most widely adopted alternative with around 52,000 GitHub stars and ships as the default ingress controller in [k3s](https://k3s.io/).

**Pros:**
- Automatic service discovery from Kubernetes
- Built-in Let's Encrypt / ACME support
- Middleware system for rate limiting, auth, headers, circuit breakers
- Comes with a dashboard for real-time monitoring
- IngressRoute CRD for advanced use cases beyond standard Ingress

**Cons:**
- Enterprise features (distributed rate limiting, WAF, OIDC) are behind a paywall
- Heavy IngressRoute CRD usage creates vendor lock-in
- Configuration can grow complex at scale

### HAProxy

[HAProxy Kubernetes Ingress Controller](https://github.com/haproxytech/kubernetes-ingress) brings over two decades of battle-tested load balancing to Kubernetes. If raw performance is what you care about most, HAProxy is hard to beat.

**Pros:**
- One of the fastest proxies available, with extremely low latency
- Advanced load balancing algorithms (least connections, source hash, URI hash)
- Proven stability. HAProxy has been in production use since 2001.
- Free and open-source community edition

**Cons:**
- Smaller Kubernetes-specific community
- Feels less cloud-native compared to newer alternatives
- Documentation for Kubernetes usage is thinner
- Fewer out-of-the-box integrations with cloud-native observability tools

### Kong

[Kong Ingress Controller](https://github.com/Kong/kubernetes-ingress-controller) is the one to look at if you need more than just routing. It's a full API gateway with authentication, rate limiting, request transformation, and a massive plugin ecosystem.

**Pros:**
- Full API gateway capabilities out of the box
- Over 100 plugins for auth, rate limiting, logging, transformations
- NGINX-based data plane, familiar territory if you're coming from nginx ingress
- DB-less mode for simpler operations

**Cons:**
- Many useful plugins require Kong Enterprise (paid)
- Heavier on resources than simpler controllers
- Overkill for straightforward routing
- Vendor-specific CRDs add operational complexity

### Contour

[Contour](https://github.com/projectcontour/contour) is an Envoy-based ingress controller that aims to be simple and Kubernetes-native. It's a CNCF incubating project with a clean separation between control plane and data plane.

**Pros:**
- Powered by Envoy, proven performance and reliability
- Simple configuration model
- HTTPProxy CRD for advanced routing beyond standard Ingress
- CNCF project with active development
- Good multi-team support through its delegation model

**Cons:**
- Smaller community compared to Traefik or Kong
- Envoy's complexity can surface when debugging
- No native Let's Encrypt support, no built-in dashboard
- Gateway API conformance is behind newer implementations

**Links**: [GitHub](https://github.com/projectcontour/contour) | [Helm Charts](https://github.com/bitnami/charts/tree/main/bitnami/contour)

### Cilium

[Cilium](https://github.com/cilium/cilium) supports the Ingress API through its eBPF-based proxy. If you're already running Cilium as your CNI, enabling ingress support is basically a configuration flag.

**Pros:**
- eBPF kernel-level processing gives you performance that's hard to match
- No separate ingress controller deployment needed (built into the CNI)
- Unified networking, security, and observability stack
- CNCF Graduated project

**Cons:**
- You need Cilium as your CNI. You can't use it as a standalone ingress controller.
- Ingress features are less mature than what dedicated controllers offer
- Troubleshooting is more involved
- If all you need is ingress, adopting a whole CNI is a big decision

### Quick Comparison

| Controller | Performance | Complexity | Unique Strength | Gateway API Ready |
|---|---|---|---|---|
| Traefik | Good | Low-Medium | Auto-discovery, Let's Encrypt | Yes |
| HAProxy | Excellent | Medium | Raw speed, 20+ years of stability | Partial |
| Kong | Good | Medium-High | API gateway + plugin ecosystem | Yes |
| Contour | Good | Low-Medium | Simple, Envoy-powered | Partial |
| Cilium | Excellent | High (CNI) | eBPF, unified networking | Yes |

## Conclusion

The retirement of nginx ingress marks the end of an era. A component that was once the universal recommendation is now archived and done.

If you're starting fresh, go with the Gateway API. Envoy Gateway if you want the best spec conformance, Traefik for ease of use, Cilium if you want the unified networking stack.

If you have a large existing Ingress setup, pick a dual-support controller like Traefik or Cilium. Swap the ingress class now, then migrate routes to Gateway API at your own pace.

If you need to stay on Ingress for now, Traefik is the most straightforward replacement. HAProxy if raw performance matters most. Kong if you need the API gateway features.

Whatever path you pick, don't sit on it. Every day you're running an unpatched ingress controller in production is a day you're accumulating risk. The project is already archived. The clock isn't ticking, it's already stopped.
