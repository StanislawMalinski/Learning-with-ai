# Kubernetes / CKA — Learning Notes

Goal: work toward the Certified Kubernetes Administrator (CKA) cert, building real understanding (not memorization).

---

## Session 1 — The Reconciliation Loop (the core principle)

### Where I'm starting from (probe results)
- I already **know behaviors**: self-healing (dead pod → replacement), Deployment is the object for a stateless rollout, and the layering **Deployment → ReplicaSet → Pods**.
- But my mental model was "schedule once, then leave it alone" — i.e. the facts were **disconnected from the principle that generates them**.
- Genuine gap: control plane internals (didn't know desired state lives in **etcd**).

### Node 1 — THE unconditional truth
> **Every actor in Kubernetes is a control loop that forever: looks at desired state, looks at actual state, and takes one step to close the gap.**

Three pieces, always:
- **Desired state** — what I declared I want (e.g. "3 replicas").
- **Actual / observed state** — what's really running now.
- **The loop** — `compare → if different, act → repeat`, forever.

The name for this is **reconciliation**. Kubernetes never "runs my commands" — it only ever *reduces the difference* between what-I-want and what-is.

### Why it HAS to be this way (derived, not decreed)
Given the environment — machines fail constantly, *after* you act, with no human watching — a one-time command can't hold a guarantee. The only design that survives is: store the **goal**, and keep re-checking reality against it. The loop is basically forced by the constraints.

### Key consequence: level-triggered, not edge-triggered
- The loop reacts to the **current gap**, not to the **event** "a pod died."
- So even if a controller is down and misses the crash event entirely, on its next observation it still sees actual=2 vs desired=3 and fixes it. No event log, no replay, no human.
- This is *the* reason self-healing is so robust: correctness depends only on eventually observing true current state.

---

## Session 1 — Node 2: "manages" = a reconciliation loop

### The decode
"Manages" always means **"runs a loop over."** There is nothing more exotic under it.
- A **ReplicaSet** *is* a loop: desired = a pod count (+ template); observed = pods that exist; action = create/delete to match.
- A **Deployment** *is* a loop one level up: desired = "a ReplicaSet running template version vN"; action = create/scale ReplicaSets to match.

### The misconception I had (important)
I first thought a ReplicaSet was a **wrapper that forwards my commands to the pods**. That's the old imperative model. It's wrong.
- **There are no commands stored or forwarded anywhere.**
- `kubectl scale rs myapp --replicas=5` does NOT send a "scale" command. It **edits one number** — the desired-state field — and saves it. The loop then sees the new gap and acts.
- I change a **noun (desired state)**, never a **verb (a command)**.
- The loop never "shuts off" when actual=desired — it keeps watching, which is how a crash is caught instantly.

### Rolling updates & rollback fall out for free
- Update image v1→v2 = edit the Deployment's template. The Deployment loop creates a **new v2 ReplicaSet**, scales it up while scaling the old v1 ReplicaSet down. That *is* a rolling update — no special procedure.
- After the update, `kubectl get rs` shows the **old ReplicaSet kept at 0 replicas** — held as a ready target so **rollback** = re-point the loop at it. (Old ones are pruned past the revision-history limit.)

### Full stack = loops nested in loops
**Deployment (loop) → ReplicaSet (loop) → Pods.** Every arrow is "manages"; every "manages" is a reconciliation loop.

---

## Session 1 — Nodes 3–5: The Control Plane

### Node 3 — the loops need a shared source of truth
Every loop is "compare desired vs actual," so state must live somewhere all loops can reliably read/write: a **shared, durable store**. (Not a command queue — there are no commands. Not private per-controller copies — they'd disagree.)

### Node 4 — etcd + API server
- **etcd**: distributed key-value database. Holds **both desired and observed state**. The **single source of truth** for the whole cluster. (CKA: you back up & restore etcd.)
- **API server**: the single **front door** to etcd. It authenticates, authorizes, validates, then reads/writes etcd.
- **IRONCLAD RULE:** *Nothing talks to etcd except the API server.* kubectl, every controller, the scheduler, every kubelet → all talk **only to the API server**. Every loop's "read/write state" = an API call.

### Node 5 — the full control plane + the flow
Components:
- **kube-controller-manager**: one process bundling dozens of controller loops (ReplicaSet, Deployment, ...).
- **scheduler**: one dedicated loop. Watches for pods with no node, picks the best node, writes `nodeName` back via the API server. **Assigns only — starts nothing.**
- **kubelet**: runs on every worker node. Watches API server for pods assigned to *its* node, tells the container runtime to pull images & run containers, and **reports actual status back up**. Where digital desired-state meets physical reality.

Full flow of "I want 3 replicas":
1. `kubectl` → **API server** → writes desired state to **etcd**.
2. **Deployment controller** sees it → creates a ReplicaSet.
3. **ReplicaSet controller** sees it → creates 3 Pod objects (no node yet).
4. **Scheduler** sees 3 unassigned pods → assigns each a node.
5. **kubelet** on each node → starts containers, reports status back.

Every arrow = a controller loop. Every read/write = through the API server. All truth = in etcd.

### Two levels of self-healing (don't confuse them)
- **Container crashes, pod object survives** → the **kubelet** restarts the container in place (per the pod's `restartPolicy`).
- **Whole pod / node lost** → the **ReplicaSet controller** sees actual < desired, creates a **new** pod → **scheduler** places it.

---

## Status
- [x] Node 1: reconciliation loop — solid.
- [x] Node 2: "manages" = a loop (Deployment→ReplicaSet→Pod) — solid.
- [x] Node 3: loops need a shared source of truth — solid.
- [x] Node 4: etcd = truth; API server = only door — solid.
- [x] Node 5: full control plane + flow — solid.

---

## Session 2 " Networking: Services

### Starting point (probe)
My model was inverted: I thought pod IPs were **pooled/stable** and that pods needed a **Service to reach each other**. Both wrong. The truth is closer to the mirror image (below).

### Node 1 " The flat network model (foundational truth)
> **Every pod gets its own unique IP, and any pod can reach any other pod directly at that IP, across all nodes, with NO NAT.**
- Like every machine on one big flat LAN having its own address. A pod does NOT share its node's IP.
- Reachability is FREE. No gateways, no per-node port maps, no same-node restriction.
- Why: avoids Docker-style port-collision hell " a pod sees the same ports inside as outside.
- Implementation is delegated to a pluggable **CNI plugin** (Calico, Cilium, ...). K8s says "make it flat"; the plugin does it. (Black box for now.)

### Node 2 " The problem Services actually solve
- Pod IPs are **ephemeral**: the ReplicaSet loop replaces pods, and each replacement gets a **new IP**.
- So Services do NOT solve *reachability* (that's free). They solve **instability**: you need one stable address for a constantly-changing set of pods.
- Wrong alternatives: permanent pod IPs (fights the model), per-request API re-query (forces every app to be K8s-aware), leader pod (single point of failure).

### Node 3 " What a Service IS
> **A Service = a stable virtual IP + DNS name in front of a set of pods. Clients hit the Service; K8s routes to a currently-live backing pod.**
- **Stable front**: the **ClusterIP** " allocated once, never changes for the Service's life. Safe to hardcode.
- **Churning back**: a continuously-reconciled list of the *actual* live pod IPs behind it.
- The client is a normal app that knows nothing about K8s " it just connects to a normal-looking IP.
- Payoff: all backing pods can be replaced (new IPs) and the ClusterIP never moves " client keeps working.

### Node 4 " Labels, selectors, Endpoints (how a Service finds its pods)
- Pods carry **labels** (key-value tags, e.g. `app: backend`).
- A Service has a **selector** (e.g. match `app: backend`). Coupling is by **label query**, never by pod identity.
- Same mechanism a ReplicaSet uses to find "its" pods. Loose coupling by label is everywhere.
- A reconciliation loop (endpoints/EndpointSlice controller) maintains the **Endpoints** object: desired = ready pods matching selector; actual = current Endpoints list; act = update as pods come/go.
- ANY ready pod matching the label joins " even a hand-made standalone pod. 
- **CKA/debug gotcha:** mismatched labels ' empty Endpoints ' 'connection refused'.

### Status (Session 2)
- [x] Node 1: flat network model
- [x] Node 2: ephemeral IPs ' the real problem
- [x] Node 3: Service = stable ClusterIP + live backing list
- [x] Node 4: labels/selectors + Endpoints (a loop)
- [x] Node 5: kube-proxy (how a packet physically reaches a pod)
- [x] Node 6: Service types (ClusterIP / NodePort / LoadBalancer)
- [x] Node 7: DNS (reach a Service by name)

### Node 5 " kube-proxy (how a packet physically reaches a pod)
- The **ClusterIP is virtual** " NO machine/interface actually owns it.
- **kube-proxy** runs on every node. It watches the API server for Services + Endpoints and programs the node's **kernel** (iptables or IPVS rules) so packets to a ClusterIP are DNAT'd to a real backing pod IP (load-balanced).
- Subtlety: kube-proxy is NOT in the packet path per-packet " it *sets up the rules*, the kernel does the rewriting at line speed.

### Node 6 " Service types (LAYERED, each a superset of the previous)
- **ClusterIP** (default): reachable only *inside* the cluster. Internal wiring.
- **NodePort**: ClusterIP + opens a high port (30000"32767) on *every node's real IP*. External traffic can get in crudely via `<node-IP>:<port>`.
- **LoadBalancer**: NodePort + asks the *cloud provider* for an external LB with a real public IP. Normal way to expose to the internet on a cloud.
- Upshot: a LoadBalancer still HAS a ClusterIP + NodePort " internal clients just use the ClusterIP, no hairpin out.

### Node 7 " DNS (reach a Service by name)
- You don't know the ClusterIP in advance / don't want to hardcode it ' use the name.
- **CoreDNS** (pods + a Service) provides cluster DNS; every pod resolves against it.
- Name scheme: `<service>.<namespace>.svc.cluster.local`. Same ns ' just `backend`; cross-ns ' `backend.production`.
- DNS resolves the name to the Service's **ClusterIP** (NOT a pod IP, NOT a NodePort).

### THE FULL CHAIN (one line)
`backend` (DNS name) ' CoreDNS ' **ClusterIP** (stable virtual IP) ' kube-proxy kernel rules ' real **pod IP** (from Endpoints, kept live by a loop) ' flat network delivers packet.
Two layers of indirection, each fixing a different instability: name'IP, and stable-IP'ephemeral-pods.

---

## Next session ideas
- Networking strand: Services (ClusterIP/NodePort/LoadBalancer), how pods talk, kube-proxy, DNS.
- Storage: Volumes, PV/PVC, StorageClasses.
- CKA hands-on: etcd backup/restore, `kubectl` imperative speed, troubleshooting a broken control-plane component.
- Namespaces, labels/selectors (how controllers actually find "their" pods).
