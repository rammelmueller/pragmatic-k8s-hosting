---
theme: ../slidev-theme-neversink/
fonts:
  sans: Roboto
  mono: Roboto Mono
  serif: Roboto Condensed
drawings:
  persist: false
mdc: true
colorSchema: light

layout: cover
transition: slide-left
---

# Self-hosting k8s
**Lukas Rammelmüller**,_TNG Technology Consulting_ <a href="https://tngtech.com" class="ns-c-iconlink"><mdi-open-in-new /></a>  


<!--
-->

---
layout: default
---

# What will we do today?

- **Quick k8s refresher** - What does kubernetes solve for us?
- **Why k8s?** - When should we use it and how?
- **Managed k8s vs. self-hosting** - Why is one harder than the other? Should we even self-host?
- **Blueprint for self hosting** - Pragmatic solutions that get the job done
- **GitOps "deepdive"** - Why and how to use pull-based config

<br>
<br>
<br>

<v-click>
... and <b>not </b> do! 

- not a k8s 101
- not a full CNCF landscape tour
- not a security / DR deep dive

</v-click>

<StickyNote  color="emerald-light" textAlign="left" width="180px"  v-drag="[632,176,304,246,5]">

<h3> After the lecture, you should have .. </h3>

- .. a clear understanding when it is viable to self-host k8s
- .. a list of standard go-to tools 
- .. the urge to use GitOps right away

</StickyNote>


<!--
Assumptions:
- You know containers and basic k8s concepts (Pods, Deployments, Services)
- Some idea of infrastructure

Goals:""
- Help you decide *whether* to self‑host k8s
- Give you a pragmatic blueprint *if* you do
-->



---
layout: two-cols-title
---


::title::

# Why containers?

::left::

- Most software today is rolled out as **images / containers**
- Why?
  - ==Portable== across environments
  - Good ==isolation== & security primitives
  - ==Convenient== build → ship → run pipeline
- But:
  - Many containers → many moving parts
  - Need orchestration, scheduling, networking, lifecycle mgmt

:: right ::

<div v-click=1>

- [Kubernetes](https://github.com/kubernetes/kubernetes) = ==orchestrator== for containers
  - Around since ~2014, CNCF since 2016
  - Wildly popular <span class='small'>(second-largest OSS project after Linux, as of 2024 71% of fortune 100 companies use it as their primary container orchestrator, ...)</span>
- What does it do for us?
  - ==Schedules== workloads on nodes
  - Handles failover <span class='small'>(restarts, health checks, rollouts, ...)</span>
  - Abstracts many otherwise complicated topics <span class='small'>(very importantly: networking!)</span>
  - Offers a rich API for infra & app configuration

</div>

::note::

\[1\] You may often hear "docker container" - same thing, misnaming for historic reasons

<!-- 
- These days most software is rolled out as images / containers
- Why? Portable, secure, convenient
- Orchestrator for containers -> Kubernetes
- Around since 2016
- What does it do for us?
  - Abstracts away many complexities (networking, failover, ...)
- Still a complex framework - not for every use-case (not everything is a nail).
  - In fact, one of the main complaints and drawbacks
  - Particularly if microservices are involved, this is at the moment one of the easiest solutions
 -->

---
layout: two-cols-title
columns: is-5
---

::title::

<h1> <img src="/img/k8s.png" width="60px"> k8s <span v-click=1 style="font-size: 3rem"> - not a magic hammer</span></h1>

:: left ::

Core k8s API refresher:

- `Namespace` – tenant / logical grouping
- `Deployment` – manages Pods, rolling updates
- `Service` – stable virtual IP / DNS for a set of Pods
- `Ingress` – HTTP(S) entry into the cluster ^1^
- `ConfigMap` / `Secret` – configuration & secret data
- `PersistentVolume` / `PersistentVolumeClaim` – storage abstraction
- ... and **many** more

::right::

<div v-click=1>

- Powerful, but ==complex==
  - Many concepts
  - Many moving parts (API server, etcd, controller manager, scheduler, …)
- Not every workload needs k8s (**YAGNI?**)
  - Simple monolith on a VM might be enough
  - Sometimes a docker-compose setup will do
- But:
  - For microservices / multi‑team ==platforms==
  - For standardized deployments
  - For hybrid / multi‑environment setups
  → k8s is often the ==least bad== option

</div>

<!-- 
Quick overview: 
- How to even interact with a cluster
- Before we move on: Make sure everyone has a fresh memory of k8s API objects

Is k8s only good?  
- Still a complex framework - not for every use-case (not everything is a nail).
  - In fact, one of the main complaints and drawbacks
  - Particularly if microservices are involved, this is at the moment one of the easiest solutions
- Despite the complexity, it is the go-to platform tool for many
  - Ecosystem is great
  - Alternative: Self-stitch together a platform that does all that for us.
  - Any self-built solution, though, inevitably resembles k8s
 - One of the greatest things though: There's a solution for many problems


> Any serious home‑grown "platform" tends to reinvent parts of k8s


 -->

---
layout: image
image: img/cncf.png
---

<StickyNote v-click=1 color="red-light" textAlign="center" width="180px"  v-drag="[383,179,215,185,5]">
<br>
<h1>There's a tool for that!</h1>

</StickyNote>

<!--
- Huge ecosystem and community
- Second‑largest OSS project after Linux
- Thousands of tools & extensions:
  - CNIs, CSIs, ingress controllers
  - Operators, GitOps controllers
  - Observability, security, policy engines

- Primary container orchestration tool for 71% of Fortune 100
- Broad vendor support (clouds, on‑prem distros)
- Dedicated meetups, conferences, SIGs
-->

---
layout: default
---

# What are we actually trying to achieve?

We want to run workloads that are:

- Stable & resilient
  - Survive node failures
  - Roll out updates safely
- Secure
  - Reasonable default posture
  - Least privilege, isolation, secrets management
- Operationally viable
  - On‑call should be bearable
  - Limited human time for maintenance
  - New people can be productive quickly

---
layout: default
---

# Constraints drive design

A "reliable system" must name its constraints:

- SLO / availability target
  - 99% (3.65 days / year) vs 99.99% (~52 mins / year)
  - Higher SLO → more complexity, more infra, more process
- RPO / RTO
  - How much data loss is acceptable?
  - How quickly must we recover?
- Team & skills
  - How many people can realistically work on infra?
- Regulatory / environment
  - Cloud OK? On‑prem / air‑gapped? Sovereign cloud?

We'll see these again when deciding managed vs self‑hosted.



---
layout: two-cols-title
---

::title::

# Managed k8s - the `easy` way

::left::

Go to your favourite cloud vendor and click:
  - AWS EKS, Azure AKS, GCP GKE
  - Or regional providers: Stackit, Hetzner, …


What you get:
- Managed control plane:
  - API server, etcd, scheduler, controllers
  - Patching, upgrades, availability
- Integration with cloud:
  - Load balancers, disks, IAM

::right::
You still own some aspects <span class='small'> (depending on the vendor)</span>

- Node sizing, OS patching (often)
- Networking layout
- Security configuration
- Workload design

---
layout: two-cols-header
---

# Managed k8s – cost perspective (qualitative)

::left::


What you pay for:

- Control plane time (cluster fee)
- Worker nodes (VMs)
- Data transfer, storage, LB, etc.

What you save:

- No etcd / control plane ops
- Less effort on upgrades & HA
- Fewer "oh no, the control plane is down at 2am" incidents

::right::

Total cost of ownership:

- \(\text{Cloud bill} + \text{Infra engineer time}\)
- One engineer day / month can be more expensive than a small control plane fee

**Rule of thumb**

- If you're already in a major cloud and don't have special constraints, managed k8s is usually cheaper overall.

---
layout: default
---

# What does "self‑managed k8s" mean?

Self‑management can mean:

- On VMs in the cloud
- On VMs you operate yourself
- On bare‑metal hardware

What unites them:

- You own the control plane lifecycle
  (install, upgrade, backup, restore)
- You own node lifecycle
- You build (some of) the "cloud magic" yourself:
  - Load balancing
  - Storage
  - Networking
  - Observability
  - Security policies & enforcement

---
layout: two-cols-header
---

# Managed vs. self‑managed 

::left::

**Managed k8s**


- Control plane:
  - Operated by provider
  - SLAs, automated upgrades
- Integrations:
  - Cloud LBs, disks, IAM
- Effort:
  - Lower baseline ops
- Downsides:
  - Vendor coupling
  - Less deep control
  - Limited in air‑gapped / special envs

::right::

**Self‑managed**

- Control plane:
  - You build + operate it
- Integrations:
  - BYO networking & storage stack
- Effort:
  - Higher ops burden
  - Need more expertise
- Upsides:
  - Full control
  - Works on‑prem / sovereign / air‑gapped

---
layout: two-cols-header
---

::left::
# When you probably should self‑host


Self‑host is reasonable if:

- You must run:
  - On‑prem
  - Air‑gapped
  - Sovereign cloud w/o managed k8s
- You need customization:
  - Special networking needs
  - Tight control over k8s internals

::right::

And you have:

- Team with infra experience
- Capacity to:
  - Design & maintain a cluster stack
  - Implement and *test* DR
  - Handle security patches, upgrades

Otherwise you risk a brittle, snowflake cluster.

---
layout: two-cols-header
---

::left::
# When you probably should not self‑host


Think twice about self‑hosting if:

- You're already on AWS/Azure/GCP
- You have a small team
- You don't have hard regulatory / locality constraints
- Your SLO is modest (e.g. 99%–99.9%)

::right::

Typical symptoms:

- "We'll figure out backups later"
- "Networking will be easy, right?"
- "We don't have time for DR tests"

> If you don't have strong reasons *for* self‑hosting,
> your default should be managed k8s.

---
layout: center
---

# From decision to reality

We assume we have good reasons to self‑host.  
What does life with a self‑hosted cluster look like?

---
layout: default
---

# Life of a self‑hosted cluster

**Day 0** – Bootstrapping

- Provision hardware / VMs
- Install OS
- Install k8s distro
- Form initial control plane & worker set

**Day 1** – First workloads

- Install CNI, ingress, cert management
- Set up observability (metrics/logs)
- Set basic security guardrails
- Deploy initial apps

**Day 2+** – Operations

- Regular upgrades (OS, k8s, addons)
- Capacity changes (scale out/in)
- Backup & DR drills
- Security updates, incident response

---
layout: default
---

# Example: new k8s version is released

What should happen in a healthy setup?

1. **Plan**
   - Review change impact
   - Decide target version & timeline
2. **Update desired state**
   - Update cluster config (e.g. distro version) in Git
3. **Rollout**
   - Control plane upgrades (one by one)
   - Worker nodes drained, upgraded, returned to service
4. **Verification**
   - Check metrics, logs, error budget
   - Run smoke tests
5. **Fallback**
   - If issues: roll back to previous version (again via Git)

We’ll see how GitOps helps orchestrate this.

---
layout: default
---

# How to make life easier?

We need to solve, at minimum:

- OS
- Kubernetes distro / flavor
- Cluster formation & lifecycle
- Networking (inside & outside)
- Storage
- Backup & recovery
- Security
- Observability

**We’ll look at pragmatic, opinionated OSS choices.**

---
layout: default
---

# OS: what do we actually need?

Two main patterns:

1. **Container‑only nodes**
   - Nodes are treated as cattle
   - Humans rarely SSH in
   - Ideal for deterministic clusters

2. **Mixed usage**
   - Example: AI clusters
   - People run experiments, scripts, fine‑tuning jobs directly on nodes
   - More drift, more risk

The OS choice follows from this.

---
layout: default
---

# OS recommendations

**Container‑only clusters**

- Prefer minimal, container‑optimized OS:
  - Fedora CoreOS
  - Talos
  - Flatcar, etc.
- Benefits:
  - Immutable / declarative
  - Easier upgrades & reproducibility

**Mixed‑use clusters**

- Use what your people know:
  - Ubuntu LTS, Debian, RHEL, …
- Be aware:
  - Higher drift and config entropy
  - Security & stability require more discipline

> If possible, separate “playground” machines from k8s nodes.

---
layout: default
---

# Kubernetes: vanilla vs distros

**Vanilla upstream k8s**

- Core components as shipped by the project
- Typically bootstrapped with `kubeadm` or similar
- You assemble surrounding tooling

**Distributions**

- Upstream k8s +:
  - Opinionated defaults
  - Installers & upgrade tooling
  - Integrations (monitoring, registry, etc.)
- Wide spectrum:
  - Lightweight (k3s, k0s, microk8s)
  - Full enterprise (OpenShift, etc.)

We’ll aim for something pragmatic in the middle.

---
layout: default
---

# Pragmatic pick: rke2

**rke2 (Rancher Kubernetes Engine 2)**

- CNCF‑conformant Kubernetes distribution
- Focus on:
  - Secure defaults (hardened configuration)
  - Simple install & upgrade
  - Works on VMs and bare‑metal

Why I like it for self‑hosting:

- Easy to get a secure cluster
- Good documentation & ecosystem
- Less boilerplate than rolling everything with bare kubeadm

Alternatives for small setups:

- k3s – lightweight, great for edge / small clusters
- microk8s – simple developer / lab environment

---
layout: default
---

# Cluster formation: from nodes to a cluster

Problem: Given some nodes, how do we:

- Form a secure cluster?
- Add nodes?
- Upgrade nodes?
- Recover a broken node or control plane?

Common approaches:

- Manual setup (good for learning, bad for prod)
- `kubeadm` + automation (Ansible, Terraform, …)
- Distro‑specific installers (e.g. rke2/k3s installers)
- Higher‑level frameworks (Cluster API, Rancher, …)
- Network boot / PXE for large fleets

---
layout: default
---

# Cluster formation: pragmatic advice

For **small to mid‑size** clusters (up to ~20 nodes):

- Use a distro with a clean bootstrap story:
  - e.g. rke2
- Wrap it with **simple automation**:
  - Cloud‑init, Ansible, or Terraform + scripts
- Keep the process:
  - Documented
  - Re‑playable
  - Versioned in Git

For **larger** clusters:

- Consider:
  - PXE / network boot
  - Centralized image mgmt
  - Cluster API or equivalent

> Don’t over‑engineer provisioning for a 5‑node cluster.  
> Don’t under‑engineer for a 200‑node fleet.

---
layout: default
---

# Networking: two domains

We need to handle:

1. **Inside the cluster**
   - Pod‑to‑Pod networking
   - Services, DNS
   - Network policies

2. **Outside to inside**
   - Client traffic into the cluster
   - Ingress, TLS termination
   - External load balancing

We’ll pick one CNI and one ingress stack and stick to them.

---
layout: default
---

# In‑cluster networking: CNI

**CNI (Container Network Interface)**

- Pluggable mechanism to provide:
  - Pod IPs
  - Routing between Pods and nodes
  - Network policies

**Pragmatic choice: Cilium**

- eBPF‑based networking
- Features:
  - Transparent encryption between nodes
  - L3/L4/L7 network policies
  - Great observability (Hubble)
- Works well on bare‑metal and cloud VMs

Other solid options: Calico, etc.  
Pick one and avoid switching later.

---
layout: default
---

# Ingress & traffic from outside

Ingredients:

- **Ingress controller**
  - NGINX Ingress Controller
  - or Traefik, or similar
- **TLS certificates**
  - `cert-manager` + ACME (Let’s Encrypt) or internal CA
- **Load balancer**
  - Cloud LB (if on cloud)
  - MetalLB / kube‑vip for bare‑metal

Pattern:

- External LB → Ingress controller → Services → Pods

We’ll later see how GitOps manages their configuration.

---
layout: default
---

# Storage: what problem are we solving?

Need to store:

- Caches / temp data for some services
- Real state:
  - Databases (Postgres, MySQL, etc.)
  - Message queues
  - User‑uploaded data

Challenges:

- Storage is **high risk, low perceived reward**
- Everyone expects it to “just work”
- When it doesn’t, the fallout is huge

We need a pragmatic approach with backups.

---
layout: default
---

# Pragmatic storage pattern

1. In‑cluster replicated storage
    - Back most PVCs with a replicated storage solution
    - Works well for:
      - Stateful apps that can tolerate some downtime
      - Smaller clusters

2. Off‑cluster backups
    - Regular backups of volumes or DBs
    - Target: object storage (S3, compatible, etc.)

3. Critical databases
    - Consider:
      - DB‑native replication + backups
      - Possibly dedicated infrastructure

---
layout: default
---

# Tool of choice: Longhorn

**Longhorn**

- CNCF project for distributed block storage on k8s
- Provides:
  - A CSI driver
  - Replication across nodes
  - Snapshots, backup/restore

How it fits:

- PVC → dynamically provisioned volume
- Longhorn → manages volumes replicated across nodes
- Backup → to external object storage

Why it's pragmatic:

- Easy to install and operate (relative to alternatives)
- Works well for small/medium self‑hosted clusters

Caveats:

- Not a silver bullet; still needs:
  - Monitoring
  - Backup strategy
  - Capacity planning

---
layout: default
---

# Backups & recovery: surfaces

We need to back up:

- **etcd**
  - Cluster state
  - API objects (Deployments, Services, etc.)
- **Persistent volumes**
  - Application data on PVs
- **Databases / external services**
  - Often best via DB‑native tools

Principle:

> You don’t have a cluster until you have a tested **restore** procedure.

---
layout: default
---

# Backups & recovery: tools and patterns

- **etcd**
  - Regular snapshots
  - Store off‑cluster
  - Practice restoring to a new control plane

- **PVCs**
  - Tools like Velero for backups & restores
  - Or CSI snapshots + backup pipeline

- **Databases**
  - Use DB‑native backup tools (pg_dump, WAL archiving, etc.)
  - Store backups off‑cluster

> “Hope is not a strategy.”

---
layout: default
---

# Security: a continuous concern

Security is not a single setup step; it's a mindset.

k8s helps with:

- Isolation primitives (namespaces, network policies)
- Pod security controls
- Service accounts & RBAC
- Integrating external identity & secret backends

We’ll look at a baseline security checklist for self‑hosting.

---
layout: default
---

# Baseline security checklist

- **RBAC**
  - Least privilege for users & service accounts
- **Pod Security**
  - Pod Security Standards (or admission policies)
  - Drop unnecessary capabilities
  - `runAsNonRoot`, no privilege escalation, seccomp, AppArmor
- **Network policies**
  - Default deny
  - Explicitly allow required traffic
- **Secrets**
  - External secret stores over plaintext k8s secrets:
    - Vault, cloud KMS + External Secrets Operator
    - SOPS‑encrypted manifests
- **Image supply chain**
  - Image scanning in CI (Trivy, Grype, …)
  - Optionally: signing (cosign), SBOMs

Use CIS benchmarks as a guide, not as dogma.

---
layout: default
---

# Observability (bonus, but crucial)

Without observability, you’re flying blind.

Minimal stack:

- **Metrics**
  - Prometheus
  - kube‑prometheus‑stack (includes node‑exporter, kube‑state‑metrics)
- **Dashboards**
  - Grafana
- **Logs**
  - Fluent Bit / vector → Loki or Elasticsearch/OpenSearch
- **Alerts**
  - Alertmanager
  - Integrate with Slack, email, PagerDuty, …

You can start simple, but you do need *something* here from Day 1.

---
layout: default
---

# Multi‑cluster (bonus)

Often you eventually need multiple clusters:

- Separation by:
  - Environment (dev / stage / prod)
  - Region / site
  - Tenant / business unit

Challenges:

- Consistent provisioning across clusters
- Managing shared components and policies
- Federated observability and identity

Tools:

- rke2 + Rancher
- Cluster API
- GitOps orchestrating multiple clusters

We’ll mostly focus on the **single cluster** picture today.

---
layout: center
---

# GitOps

Making k8s operations and deployments
boring, repeatable and auditable

---
layout: default
---

# Control loops & reconciliation

Core k8s idea:

- You declare **desired state** (YAML, manifests, CRs)
- Controllers continuously reconcile:
  - Compare desired vs actual
  - Take actions to converge actual → desired

GitOps extends this idea:

- Desired state lives in **Git**
- A controller reconciles **cluster state** to what’s in Git

---
layout: default
---

# What is GitOps?

- **Git as single source of truth** for:
  - Cluster configuration (CNI, ingress, certs, observability)
  - Security policies
  - Application deployments
- **Automated reconciliation**
  - A controller monitors Git
  - Applies changes to the cluster
  - Continuously keeps cluster in sync

Why it’s useful:

- Every change is a PR (reviewable, auditable)
- Rollback = `git revert`
- Reproducible environments
- Less “kubectl from my laptop at 2am”

---
layout: default
---

# Why GitOps from day 0?

Common misconception:

> “We’ll just add GitOps later.”

Problems with that:

- You accumulate **snowflake state** in the cluster
- Migration to GitOps later:
  - Means reconciling state drift
  - Is painful and time‑consuming

Better approach:

- Start **simple** with GitOps:
  - Maybe 1 repo, minimal patterns
- Grow patterns as you need them

---
layout: default
---

# GitOps tools

Two dominant options:

- **Argo CD**
  - App‑centric
  - Great UI and visualization
  - Supports “app of apps” pattern
- **Flux**
  - Git‑native feel
  - CRD‑driven
  - Lightweight, easy to integrate into Git workflows

Both:

- Continuously reconcile from Git to cluster
- Support Helm, Kustomize, raw manifests
- Are CNCF projects, battle‑tested

Pick one, don’t try to run both.

---
layout: default
---

# Typical GitOps repo structure

One possible pattern (simplified):

- `clusters/`
  - `prod/`
  - `staging/`
  - `dev/`
- `infrastructure/`
  - CNI, ingress controller, cert‑manager
  - Longhorn, monitoring, logging, security policies
- `apps/`
  - Individual applications (Helm charts / manifests)
- `environments/`
  - Overlays for dev/stage/prod (kustomize or Helm values)

This keeps:

- Infra vs apps clearly separated
- Environment‑specific differences localized

---
layout: default
---

# How GitOps ties the stack together

Everything we’ve discussed can be **described in Git**:

- rke2 cluster config
- Cilium, ingress controller, cert‑manager
- Longhorn + backup jobs
- Observability stack (Prometheus, Grafana, Loki)
- Security policies (RBAC, PSS, network policies)
- Application deployments & configs

GitOps controller:

- Watches these repos
- Applies changes
- Keeps your cluster in the desired state

---
layout: default
---

# Without GitOps vs with GitOps

**Without GitOps**

- `kubectl apply` from laptops
- Imperative `helm upgrade` commands
- Hotfixes nobody documented
- Hard to reproduce a cluster
- “Works on prod. Don’t touch it.”

**With GitOps**

- All changes via PRs
- `main` branch defines cluster state
- Reverts roll back changes
- Easy to spin up a new cluster with the same config
- Clear audit trail of *who* changed *what* and *when*

---
layout: default
---

# Example: upgrading ingress controller

**Without GitOps**

1. Engineer edits values file locally
2. Runs `helm upgrade` from laptop
3. Something breaks
4. Nobody remembers exactly what changed
5. Rollback is manual and stressful

**With GitOps**

1. PR updates ingress Helm chart version in Git
2. GitOps controller rolls out change
3. Observability indicates health
4. If issues:
   - Revert commit
   - Controller rolls back
5. Everything recorded in Git history

---
layout: default
---

# GitOps doesn’t solve everything

GitOps helps with:

- Declarative config
- Rollouts and rollbacks
- Auditability and drift detection

It does **not** magically solve:

- Environment promotion semantics
  - e.g. promoting artifacts from dev → stage → prod
- Release orchestration across multiple services
- All security concerns

Complementary tools/patterns:

- Promotion controllers (e.g. Kargo)
- Workload tagging / artifact versioning
- Policy as code (OPA/Gatekeeper, Kyverno, …)

---
layout: center
---

# Summary & decision framework

---
layout: two-cols-header
---

::left::

# Pragmatic self‑hosted stack (cheat sheet)


**Infra & OS**

- Container‑only nodes:
  - Fedora CoreOS / Talos / similar
- Mixed use:
  - Ubuntu LTS / Debian / RHEL

**Kubernetes**

- Distro: rke2
- Small edge clusters: k3s

**Networking**

- CNI: Cilium
- Ingress: NGINX or Traefik
- TLS: `cert-manager`
- Bare‑metal LB: MetalLB / kube‑vip

::right::

**Storage & backup**

- Storage: Longhorn
- Backups:
  - etcd snapshots
  - Velero (or similar) for PVs
  - DB‑native backups + off‑cluster storage

**Security**

- RBAC & Pod Security Standards
- Network policies (default deny)
- External secrets (Vault, SOPS, …)
- Image scanning in CI

**Observability & GitOps**

- Observability:
  - kube‑prometheus‑stack + Grafana
  - Loki for logs
- GitOps:
  - Argo CD or Flux

---
layout: default
---

# Honest trade‑offs

**Advantages of self‑hosting**

- Full control over:
  - k8s version and config
  - Networking & storage stack
  - Security tooling
- Works in:
  - On‑prem, air‑gapped, sovereign environments

**Costs**

- Operational burden:
  - Upgrades, DR, security patches
  - On‑call for control plane issues
- Complexity:
  - Networking, storage, backup, security all in your lap
- Human cost:
  - Need infra expertise and time, continuously

---
layout: default
---

# Simple decision rules

- On a major cloud, no strong constraints?
  - **Default:** managed k8s
- Must run on‑prem / air‑gapped / sovereign?
  - Self‑hosting is often justified
- Team size & skills:
  - If you cannot afford at least some dedicated infra capacity (and regular DR tests), think twice
- SLO:
  - The higher your SLO, the more discipline you need:
    - GitOps
    - Observability
    - Tested backups and DR

Self‑hosting is powerful, but should be a conscious choice.

---
layout: center
---

# Q & A

Happy to go deeper into:

- Storage and backups
- GitOps patterns
- Security hardening
- Specific tools (rke2, Cilium, Longhorn, …)
