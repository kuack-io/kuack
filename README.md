<div align="center">
  <img src="static/logo-small.svg" alt="Kuack Logo" width="120" />
</div>

# Kuack

> "If it looks like a Node, and it quacks like a Node... It's a Kuack Node!" 🦆

**Kuack** is a Virtual Kubelet provider that transforms browsers into Kubernetes worker nodes, enabling distributed computing at the edge using WebAssembly.

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![GitHub](https://img.shields.io/github/stars/kuack-io/kuack?style=social)](https://github.com/kuack-io/kuack)

## What is Kuack?

Kuack enables Kubernetes to schedule WebAssembly (WASM) workloads to browser-based agents. Instead of running pods on expensive cloud servers, Kuack can execute them in users' browsers - turning millions of idle devices into a massive compute cluster.

### Key Features

- **Virtual Kubelet Integration**: Works with standard Kubernetes APIs
- **Multi-Platform Support**: Same image runs on Linux nodes and browser agents
- **Fallback to Regular Nodes**: Pods can automatically failover to regular cluster nodes if no browser agents are available
- **Container2Wasm Support**: Run existing containers converted to WASM (see [c2w-examples](https://github.com/kuack-io/c2w-examples))
- **Zero Installation**: Works in any modern browser
- **Resource Aggregation**: Sums CPU, memory, and GPU from connected agents

## Architecture

![](static/architecture.jpg)

## Quick Start

For a full walkthrough, see the [Quickstart](https://kuack.io/docs/quickstart) in the docs. At a high level:

1. **Install Kuack via Helm:**

   ```bash
   helm install kuack oci://ghcr.io/kuack-io/charts/kuack --wait
   ```

2. **Expose Node and Agent services locally (for a quick test):**

    Node:

    ```bash
    kubectl port-forward service/kuack-node 8081:8080
    ```

    Agent:

    ```bash
    kubectl port-forward service/kuack-agent 8080:8080
    ```

3. **Connect a browser agent:**

    Open <http://localhost:8080> in your browser. The Agent UI should appear. Point it at the Node WebSocket endpoint, for example `ws://127.0.0.1:8081` when using the port-forwards above. Use default token value to connect.

    > **Note**: This standalone Agent UI is a **demonstration** app. In a real production scenario, you would embed the Kuack Agent library directly into your own frontend application, so your users connect automatically when they visit your site.

4. **Run the checker example Pod:**

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: checker
    spec:
      nodeSelector:
        kuack.io/node-type: kuack-node
      tolerations:
        - key: "kuack.io/provider"
          operator: "Equal"
          value: "kuack"
          effect: "NoSchedule"
      containers:
        - name: checker
          image: ghcr.io/kuack-io/checker:latest
          env:
            - name: TARGET_URL
              value: "https://kuack.io"
    ```

    Apply it and then stream logs:

    ```bash
    kubectl apply -f checker.yaml
    kubectl logs -f checker
    ```

This uses a multi-arch image that can run both on regular nodes and in browsers.

**To see both worlds in action:**
1.  Keep the `nodeSelector` to force execution on the browser (Kuack node).
2.  Remove `nodeSelector` and `tolerations` to let Kubernetes schedule it on a regular node (if available).
3.  **The Killer Feature**: Use `affinity` (or just rely on scheduler scoring) to prefer browsers but fall back to regular nodes.

```yaml
    affinity:
      nodeAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            preference:
              matchExpressions:
                - key: kuack.io/node-type
                  operator: In
                  values:
                    - kuack-node
```

**Note:** Log streaming (`kubectl logs`) will not work in K3s clusters because its implementation for kubelet connectivity (using a custom remotedialer tunnel) is non-standard. Support for K3s will be added in later releases. For now, please use Minikube, Kind, EKS, GKE, or other non-Rancher clusters.

## How It Works

### 1. Agent Connection

### 1. Agent Connection

Browser agents connect to the Kuack Node via WebSocket.
*   **Demo**: You open the standalone Agent UI page.
*   **Production**: The Agent library is running in background on your users' existing tabs (e.g., in your web app), connecting automatically.

### 2. Resource Aggregation

The Node sums resources from all connected agents and presents them to Kubernetes as a virtual node.

### 3. Pod Scheduling

Kubernetes schedules WASM-compatible pods to the virtual node. The Node selects an available agent.

### 4. WASM Execution

The agent downloads the WASM binary from the Registry Proxy and executes it in the browser.

### 5. Results

Execution results and logs stream back to Kubernetes via the Node.

## Integration

### Production: Embed the Agent

For production, you don't ask users to open a separate "Kuack Agent" page. Instead, you integrate the agent library into your existing web application.

*   **Zero friction**: Users contribute capacity just by browsing your site.
*   **Seamless**: The agent runs in a WebWorker in the background.

*(Integration documentation coming soon)*

## Run Standard Containers (Container2Wasm)

One of Kuack's most powerful features is the ability to run **standard OCI images** (like Python, Node.js, Ubuntu, Busybox, etc.) in the browser.

We use [container2wasm](https://github.com/ktock/container2wasm) to convert x86_64 containers into WASM/WASI images. These images are multi-arch: they run natively on Linux nodes and as WASM in the browser agent! You can find ready-to-use images in our [c2w-examples](https://github.com/kuack-io/c2w-examples) repository or create your own.

### Examples

You can try these ready-to-use images immediately. They are hosted in our [c2w-examples](https://github.com/kuack-io/c2w-examples) repository.

<details>
<summary><strong>Python</strong></summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: python-c2w
spec:
  nodeSelector:
    kuack.io/node-type: kuack-node
  tolerations:
    - key: "kuack.io/provider"
      operator: "Equal"
      value: "kuack"
      effect: "NoSchedule"
  containers:
    - name: python
      image: ghcr.io/kuack-io/c2w-examples/python:3.14-alpine
      command: ["python3", "-c", "print('Hello from Python in the Browser!')"]
```

</details>

<details>
<summary><strong>Node.js</strong></summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-c2w
spec:
  nodeSelector:
    kuack.io/node-type: kuack-node
  tolerations:
    - key: "kuack.io/provider"
      operator: "Equal"
      value: "kuack"
      effect: "NoSchedule"
  containers:
    - name: node
      image: ghcr.io/kuack-io/c2w-examples/node:24-alpine
      command: ["node", "-e", "console.log('Hello from Node.js in the Browser!')"]
```

</details>

<details>
<summary><strong>Ubuntu</strong></summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-c2w
spec:
  nodeSelector:
    kuack.io/node-type: kuack-node
  tolerations:
    - key: "kuack.io/provider"
      operator: "Equal"
      value: "kuack"
      effect: "NoSchedule"
  containers:
    - name: ubuntu
      image: ghcr.io/kuack-io/c2w-examples/ubuntu:24.04
      command: ["/bin/bash", "-c", "cat /etc/os-release"]
```

</details>

*Note:* Remember that networking is currently restricted in browser-based executions.

## Ecosystem

Kuack consists of multiple components:

- **[node](https://github.com/kuack-io/node)**: Virtual Kubelet provider (Go)
- **[agent](https://github.com/kuack-io/agent)**: Browser agent library (TypeScript)
- **[helm](https://github.com/kuack-io/helm)**: Helm chart for deployment
- **[checker](https://github.com/kuack-io/checker)**: Example WASM application (Rust)
- **[docs](https://github.com/kuack-io/docs)**: Documentation repository
- **[e2e](https://github.com/kuack-io/e2e)**: End-to-end test suite (Cucumber, BDD)
- **[c2w-examples](https://github.com/kuack-io/c2w-examples)**: Example container images converted to WASM (Python, Node.js, Ubuntu, etc.)
- **[devspace](https://github.com/kuack-io/devspace)**: Local development setup for Kuack

## Limitations

Kuack is designed for specific workload types:

- Stateless workloads
- CPU-intensive tasks
- Short-lived jobs
- WASM-compatible code

It's **not an option** for:

- Stateful services (databases, caches)
- Long-running services
- Privileged operations
- Network services (listening on ports)

### Specific Limits

- **Environment Variables**: Due to WASI constraints, the total size of environment variables (IOV_MAX) is limited to **1024 bytes**. Keep env vars concise.

See [Limitations](https://kuack.io/docs/browser-runtime) for complete details.

## Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for information about the development process and tools available to simplify development.

Areas where help is needed:

- Documentation improvements
- Additional example applications, including [container2wasm](https://github.com/container2wasm/container2wasm) cases
- Bug fixes and stability improvements
- End-to-end tests for our [e2e](https://github.com/kuack-io/e2e) suite

## Roadmap

We have ambitious plans to take Kuack from prototype to production-grade:

- **Registry Separation**: Extracting the internal registry into a standalone, scalable service (currently in-memory).
- **Distributed Nodes**: Moving from a single-node architecture to a horizontally scalable, distributed system (likely using Redis for coordination).
- **Persistence**: Implementing caching layer for WASM binaries and logs (currently in-memory).
- **Log Collection**: Adding sidecar support to run reliable log collectors like Fluentbit or Promtail on the virtual node.
- **High Concurrency**: Optimizing for millions of concurrent agent connections.

## License

Licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) for details.

## Links

- **Website**: <https://kuack.io>
- **Documentation**: <https://kuack.io/docs>
- **Helm Chart**: <https://github.com/kuack-io/helm>

## Status

🚧 **Early Development**: Kuack is currently a proof of concept. Not recommended for production workloads yet.

## Why Kuack?

The name combines "Kubernetes" (K8s) with "Quack" (duck sound), referencing the "duck test": *"If it looks like a duck and quacks like a duck, it's probably a duck."* In our case: if it looks like a Kubernetes node and behaves like a Kubernetes node, it's a Kuack node!

---

**Made with 🦆 by the Kuack team**
