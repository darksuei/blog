+++
title = 'KubeRPC v2: Actually Kubernetes-Native'
date = '2026-06-28T00:00:00.000Z'
description = 'KubeRPC v2 now ships a mutating admission webhook that eliminates all in-cluster configuration. Your services just work with a single annotation.'
author = 'Folarin Raphael'
readTime = true
tags = [
  "kubernetes",
  "rpc",
  "grpc",
  "cloud native",
  "microservices",
  "distributed systems",
  "service discovery",
  "platform engineering",
  "infrastructure engineering",
  "software engineering",
  "golang",
  "developer tools",
  "kuberpc"
]
+++

About a two years ago, I wrote about [KubeRPC](https://github.com/darksuei/kubeRPC). A lightweight RPC framework I had built to reduce the latency overhead of service-to-service communication inside a Kubernetes cluster.

The core idea was simple, instead of going through the entire HTTP request/respone pipeline for every internal call, services can talk to each other directly over persistent TCP connections with MessagePack encoding. Benchmarks showed roughly 60% lower latency compared to equivalent HTTP calls in the same cluster.

But there was a problem.

**What was the problem?**

Services that wanted to participate had to be configured manually, pass in the core URL, the service name, the host, the port. Every. Single. Time.

```js
const rpc = new KubeRPC({
  coreURL: "http://kuberpc-core:8080",
  serviceName: "user-service",
  host: "user-service.default.svc.cluster.local",
  port: 7749
});
```

Every developer working on a service needed to know about KubeRPC's internals just to use it. That is the opposite of seamless integration, and quite non-developer friendly.

The framework was simply Kubernetes-aware, but not Kubernetes-native.

**What changed in v2?**

v2 ships a [mutating admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/). If you are not familiar, this is a Kubernetes mechanism that lets you intercept pod creation requests and modify them before the pod gets scheduled.

When the KubeRPC sees a pod annotated with `kuberpc.suei.io/enabled: "true"`, it patches the pod spec with the configuration it needs, injected directly as environment variables before the container even starts.

The same setup from above now looks like this:

```yaml
annotations:
  kuberpc.suei.io/enabled: "true"
  kuberpc.suei.io/service: "user-service"
```

And on the SDK side:

```js
const rpc = new KubeRPC();
```

No arguments. No knowledge of the core registry URL. No hardcoded hostnames. The webhook simply handles it.

**What exactly does the webhook inject?**

Four environment variables:

- `KUBERPC_CORE_URL` - address of the core service registry
- `KUBERPC_SERVICE_NAME` - your service name, from the annotation
- `KUBERPC_HOST` - the cluster DNS name, managed by kubeRPC (`<service>.<namespace>.svc.cluster.local`)
- `KUBERPC_PORT` - the TCP port the SDK listens on (default: 7749)

The SDK reads these on startup and configures itself. No code changes required when you move between environments. No config files to manage. Adding a new service is now as simple as adding a few annotations on your deployment spec.

**What about calling other services?**

v2 also ships a much cleaner interface. Previously:

```js
const result = await rpc.call("user-service", "getUser", { id: "123" });
```

Now:

```js
const userService = rpc.service("user-service"); // Service proxy for the service that owns the method to be called
const user = await userService.call("getUser", { id: "123" });
```

Subtle but major difference, you create a proxy once per dependency and use it like any other object. The underlying endpoint is resolved and cached on first use, every call after that goes directly over the persistent TCP connection with no registry lookup.

**How does that TCP connection actually work?**

On the first call to a service, the SDK asks the core registry for the target's host and port, then opens a TCP connection. That connection stays open. Every subsequent call skips the registry entirely. Now it's just a MessagePack frame going over the wire and a response coming back.

The core is consulted exactly once per service, per process lifetime.

**Does the webhook break anything if it goes down?**

No. The webhook is configured with `failurePolicy: Ignore`. If the webhook is unavailable for any reason, pod scheduling continues as normal. Services without the annotation are completely unaffected. KubeRPC never sits in the critical path of your cluster.

**Getting started**

Deploy the core with Helm. The webhook and its TLS certificates are bundled and managed automatically by the chart.

```bash
helm upgrade --install kuberpc-core \
  oci://ghcr.io/darksuei/charts/kuberpc-core \
  --version 2.0.0 \
  -n kuberpc \
  --create-namespace \
  --wait
```

Annotate your pods:

```yaml
annotations:
  kuberpc.suei.io/enabled: "true"
  kuberpc.suei.io/service: "my-unique-service-name"
```

Install the SDK:

```bash
npm install kuberpc-node
```

Register methods on your server:

```js
import { KubeRPC } from "kuberpc-node";

const rpc = new KubeRPC();

await rpc.register("getUser", async ({ id }) => {
  return db.users.findById(id);
});
```

Call them from any other service:

```js
import { KubeRPC } from "kuberpc-node";

const rpc = new KubeRPC();
const userService = rpc.service("user-service");
const user = await userService.call("getUser", { id: "123" });

rpc.close();
```

**What is still on the roadmap?**

The foundation is solid. What I am still working on:

- Request multiplexing: Concurrent calls to the same service are currently serialised through a promise queue. Adding request IDs to the wire format will allow them to travel in parallel over a single connection.
- Retries and timeouts.
- Go and Python SDKs: The protocol is language-agnostic, SDKs would be available in different languages.
- Service TTL and heartbeat: Stale registrations should expire automatically.
- Observability and Tracing.

**Give it a shot**

If your services are already running in Kubernetes, you are two annotations away from dropping HTTP for all internal calls.

[KubeRPC on GitHub](https://github.com/darksuei/kubeRPC) | [npm](https://www.npmjs.com/package/kuberpc-node)
