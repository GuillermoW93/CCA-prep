
# Cilium Lab on macOS with OrbStack

A small local Kubernetes lab for practicing Cilium, Hubble, Hubble UI, endpoint and identity inspection, network monitoring, connectivity testing, and Kubernetes NetworkPolicy enforcement.

## Lab components

- **OrbStack** – local container runtime
- **Kind** – single-node Kubernetes cluster
- **Cilium** – Kubernetes CNI
- **Hubble** – network-flow observability
- **Hubble UI** – graphical network-flow visualization
- **kubectl** – Kubernetes management

This guide assumes OrbStack is installed and running on macOS and Homebrew is available.

---

## 1. Install the command-line tools

```bash
brew install kubectl
brew install kind
brew install helm
brew install cilium-cli
brew install hubble
```

Check that OrbStack is running:

```bash
orb status
docker info
```

Inspect Docker contexts:

```bash
docker context ls
```

If `orbstack` is listed, select it:

```bash
docker context use orbstack
```

Verify the tools:

```bash
kubectl version --client
kind version
helm version
cilium version
hubble version
```

---

## 2. Create a Kind cluster

Create a working directory:

```bash
mkdir -p ~/cilium-lab
cd ~/cilium-lab
```

Create `kind-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

networking:
  disableDefaultCNI: true

nodes:
  - role: control-plane
```

Create the cluster:

```bash
kind create cluster \
  --name cilium-lab \
  --config kind-config.yaml
```

Check the cluster:

```bash
kubectl cluster-info
kubectl get nodes
```

The node may initially show `NotReady`. This is expected because no CNI has been installed yet.

---

## 3. Install Cilium

Install Cilium:

```bash
cilium install \
  --version 1.20.4 \
  --set ipam.mode=kubernetes
```

Wait for Cilium to become operational:

```bash
cilium status --wait
```

Inspect the system pods:

```bash
kubectl get pods -n kube-system
```

Try the first Cilium commands:

```bash
cilium status --wait
cilium config view
cilium endpoint list
cilium identity list
```

---

## 4. Enable Hubble and Hubble UI

Enable Hubble with its UI:

```bash
cilium hubble enable --ui
```

Check Cilium status:

```bash
cilium status
```

Inspect the system pods:

```bash
kubectl get pods -n kube-system
```

You should see Hubble Relay and Hubble UI components running.

### Open Hubble UI

Run:

```bash
cilium hubble ui
```

This starts a local port-forward and normally opens the UI in your browser.

If it does not open automatically, navigate to:

**http://localhost:12000**

Keep the command running while using the UI.

Press `Ctrl+C` to stop port forwarding.

### Use the Hubble CLI

Check Hubble:

```bash
hubble status
```

Observe flows:

```bash
hubble observe
```

Observe flows in the `default` namespace:

```bash
hubble observe --namespace default
```

Shorter equivalent:

```bash
hubble observe -n default
```

---

## 5. Create workloads

Create a backend Deployment:

```bash
kubectl create deployment backend \
  --image=nginx
```

Expose it as a Service:

```bash
kubectl expose deployment backend \
  --port=80 \
  --target-port=80
```

Create a frontend pod:

```bash
kubectl run frontend \
  --image=curlimages/curl \
  --command -- sleep 36000
```

Wait for the workloads:

```bash
kubectl get pods -o wide
kubectl get services
```

Check pod labels:

```bash
kubectl get pods --show-labels
```

The backend Deployment should have the label:

```text
app=backend
```

Generate traffic from frontend to backend:

```bash
kubectl exec frontend -- \
  curl -s http://backend
```

You should receive the default Nginx welcome page HTML.

---

## 6. Practice the Cilium commands

### Check Cilium health

```bash
cilium status --wait
```

### View Cilium configuration

```bash
cilium config view
```

### List Cilium endpoints

```bash
cilium endpoint list
```

Endpoint IDs can vary. Use an ID shown by the previous command:

```bash
cilium endpoint get <ENDPOINT_ID>
```

### List security identities

```bash
cilium identity list
```

### Monitor network events

Run this from your Mac. It executes the debug command inside the Cilium agent pod:

```bash
kubectl -n kube-system \
  exec -it ds/cilium -- \
  cilium-dbg monitor
```

In another Terminal, generate traffic:

```bash
kubectl exec frontend -- \
  curl -s http://backend
```

### Run connectivity tests

```bash
cilium connectivity test
```

This creates temporary test workloads and can take several minutes.

### Observe Hubble flows

```bash
hubble observe
```

Filter by namespace:

```bash
hubble observe --namespace default
```

Or use:

```bash
hubble observe -n default
```

Follow new events continuously:

```bash
hubble observe -n default --follow
```

Filter forwarded traffic:

```bash
hubble observe --verdict FORWARDED
```

Filter TCP traffic:

```bash
hubble observe --protocol TCP
```

### Inspect egress gateway BPF entries

Run the debug command inside the Cilium agent pod:

```bash
kubectl -n kube-system \
  exec -it ds/cilium -- \
  cilium-dbg bpf egress list
```

An empty result is normal in this basic lab because an egress gateway has not been configured.

---

## 7. Explore traffic in Hubble UI

With the UI port-forward running, open:

**http://localhost:12000**

Generate traffic from frontend:

```bash
kubectl exec frontend -- \
  curl -s http://backend
```

In Hubble UI:

1. Select or filter to the `default` namespace.
2. Inspect frontend and backend communication.
3. Open individual flows to inspect their details.
4. Compare the graphical information with the output of `hubble observe -n default`.

You can leave the UI open while running the NetworkPolicy exercise below.

---

## 8. Understand the Hubble output

A flow may look similar to this:

```text
default/frontend:55106 (ID:43582) -> default/backend-74769dbc96-8kk2w:80 (ID:45968) to-endpoint FORWARDED (TCP Flags: SYN)
```

Useful fields:

| Field | Meaning |
|---|---|
| `default` | Kubernetes namespace |
| `frontend` | Source pod |
| `55106` | Source port |
| `ID:43582` | Source Cilium security identity |
| `backend-...` | Destination pod |
| `80` | Destination port |
| `ID:45968` | Destination Cilium security identity |
| `to-endpoint` | Traffic delivered toward the destination endpoint |
| `FORWARDED` | Flow was forwarded |
| `SYN`, `ACK`, `FIN`, `PSH` | TCP flags |

You may also see traffic from frontend to:

```text
kube-system/coredns-...:53
```

This is DNS traffic generated when frontend resolves the Kubernetes Service name `backend`.

---

## 9. Exercise: block backend ingress with NetworkPolicy

This exercise blocks all ingress traffic to pods selected by `app=backend`.

It demonstrates Kubernetes NetworkPolicy enforcement and lets you inspect dropped traffic in Hubble CLI and Hubble UI.

### Create deny-backend.yaml

Create a file named `deny-backend.yaml` in `~/cilium-lab`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
```

### Check the backend label

```bash
kubectl get pods --show-labels
```

Confirm that the backend pod has:

```text
app=backend
```

### Apply the policy

```bash
kubectl apply -f deny-backend.yaml
```

Verify that it exists:

```bash
kubectl get networkpolicy -n default
```

### Watch Hubble while testing

In one Terminal, follow flows:

```bash
hubble observe -n default --follow
```

In another Terminal, try to reach the backend:

```bash
kubectl exec frontend -- \
  curl --max-time 5 http://backend
```

Expected result:

The request should fail or time out because the policy selects the backend pods and allows no ingress traffic.

Hubble should show dropped traffic associated with the denied connection.

You can also inspect the flows in Hubble UI:

**http://localhost:12000**

Look for the `DROPPED` verdict.

> Important: This policy denies all ingress traffic to the selected backend pods, not just traffic from frontend. It does not block frontend DNS requests to CoreDNS.

### Restore connectivity

Delete the policy:

```bash
kubectl delete networkpolicy deny-backend
```

Test connectivity again:

```bash
kubectl exec frontend -- \
  curl -s http://backend
```

The Nginx welcome page should be returned again.

---

## 10. Clean up

Delete the Kind cluster:

```bash
kind delete cluster --name cilium-lab
```

This removes the Kubernetes cluster and its workloads.

The `~/cilium-lab` directory and configuration files remain on your Mac.

To recreate the lab later, return to the directory and repeat the cluster creation and installation steps.

---

## Command quick reference

| Command | What it does |
|---|---|
| `cilium status --wait` | Waits for Cilium to become operational |
| `cilium config view` | Displays Cilium configuration |
| `cilium endpoint list` | Lists Cilium endpoints |
| `cilium identity list` | Lists security identities |
| `cilium-dbg monitor` | Observes live network events |
| `cilium connectivity test` | Runs connectivity tests |
| `hubble observe` | Inspects network flows |
| `cilium hubble enable --ui` | Enables Hubble and its UI |
| `cilium hubble ui` | Starts local port-forwarding for Hubble UI |
| `cilium-dbg bpf egress list` | Inspects egress gateway BPF entries |

---

## Notes

- This is a learning lab, not a production Kubernetes configuration.
- The commands use Cilium version `1.20.4` in the installation step. If that version is no longer available, choose a currently supported version from the [Cilium installation documentation](https://docs.cilium.io/en/stable/installation/).
- `cilium-dbg` is available inside the Cilium agent container in this setup. That is why the examples invoke it with `kubectl exec`.
- The egress gateway BPF map may be empty until an egress gateway is configured and used.
- Keep the Hubble UI port-forward command running in its Terminal for as long as you want to use the UI.
- The NetworkPolicy exercise assumes the backend pods carry the `app=backend` label.

