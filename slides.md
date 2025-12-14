---
theme: ../../slidev-theme-neversink/
fonts:
  sans: Roboto
  mono: Roboto Mono
  serif: Roboto Condensed
drawings:
  persist: false
mdc: true
colorSchema: light
color: blue

layout: cover
transition: slide-left
---

# Self-hosting `k8s`

<div style="margin-top: -2rem">

The _pragmatic_ way

</div>

<img src="/img/k8s.png"  v-drag="[412,135,687,668]">

<img src="/img/TNG-Logo_groß_mitSchutzzone_weiß.svg"  v-drag="[745,-53,215,186]">

::note::
**Lukas Rammelmüller**,_TNG Technology Consulting_ <a href="https://tngtech.com" class="ns-c-iconlink"><mdi-open-in-new /></a>

---
layout: default
---

# What will we do today?

- ==Quick k8s refresher== - What does kubernetes solve for us?
- ==Why k8s?== - When should we use it and how?
- ==Managed k8s vs. self-hosting== - Why is one harder than the other? Should we even self-host?
- ==Blueprint for self hosting== - Pragmatic solutions that get the job done
- ==GitOps "deepdive"== - Why and how to use pull-based config
- ... and we'll set up our **own cluster** too!

<br>
<br>
<br>

<v-click>
... and <mark class='red'>not</mark> do! 

- not a k8s 101!
- not a full CNCF landscape tour!
- not a security / desaster recovery deep dive!

</v-click>

<StickyNote  color="emerald-light" textAlign="left" width="180px"  v-drag="[632,176,304,246,5]">

<h3> After the lecture, you should have .. </h3>

- .. a clear understanding when it is viable to self-host k8s
- .. a list of standard go-to tools 
- .. the urge to use GitOps right away
- .. some hands on experience with that!

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

- `Namespace` - tenant / logical grouping
- `Deployment` - manages Pods, rolling updates
- `Service` - stable virtual IP / DNS for a set of Pods
- `Ingress` - HTTP(S) entry into the cluster ^1^
- `ConfigMap` / `Secret` - configuration & secret data
- `PersistentVolume` / `PersistentVolumeClaim` - storage abstraction
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
layout: two-cols-title
---

::title::

# What do we want - and at what cost?

::left::

### Requirements for our workloads

- Stable & resilient
  - Survive node failures
  - Roll out updates safely
- Secure
  - Reasonable defaults
  - Least privilege, isolation, secrets management
- Operationally viable
  - On‑call should be bearable
  - Possibility to react (fail) fast
  - "Standardized setup" -> Onboard people to be productive quickly


::right::

### Constraints drive design!

- ==Service Level Objective (SLO)== / availability target
  - 99% (3.65 days / year) vs 99.99% (~52 mins / year)
  - Higher SLO → more complexity, more infra, more process
- ==Revovery== from failures
  - How much data loss is acceptable? (RPO)
  - How quickly must we recover? (RPT)
- ==Team & skills==
  - How many people can realistically work on infra?
- ==Regulatory== / environment
  - Cloud OK? On‑prem / air‑gapped? Sovereign cloud?




---
layout: two-cols-title
---

::title::

# Managed k8s - the "easy" way

::left::

Go to your favourite ==cloud vendor== and click (or use IaC tools)
  - AWS EKS, Azure AKS, GCP GKE, …
  - 🇪🇺 providers: Stackit, Hetzner, …


What we get:
- Managed control plane:
  - API server, etcd, scheduler, controllers
  - High availability
  - Patching + upgrades
- Integration with cloud
  - Load balancers, storage, IAM
  - Logging and monitoring
  - Backups

::right::

You still have to take care of some aspects <span class='small'> (depending on the vendor)</span>
- Manage worker nodes (sizing, scaling)
- Security configuration
- Workload delivery



<StickyNote v-drag="[515,284,353,177,-5]" text-align=center color="emerald-light">
<br>

### Essentially we get a k8s API endpoint that we can use to deploy our applications - all inclusive, hassle-free

</StickyNote>

---
layout: two-cols-title
---

::title::

# Option 2 - self-manage k8s

::left::

### Self‑management can mean ...
- ... VMs in the cloud
- ... VMs you operate yourself
- ... bare‑metal hardware


### What unites them
- You own the control plane lifecycle
  (install, upgrade, backup, restore)
- You own node lifecycle
- You build (some of) the ==cloud magic== yourself:
  - Load balancing
  - Storage
  - Networking
  - Observability
  - Security policies & enforcement
  - Public key infrastructure (PKI)


::right::

### Benefits

- Ability to run under ==all conditions==
  - highly regulated environments
  - compliance / privacy concerns
  - edge
- Full control over
  - infrastructure: allows full customization
  - data: what is stored where?
- **No vendor lock-in!**
- Side bonus: Build capabilities that are important when the <mark class='red'>incidents</mark> happen  <span class='small'> (and they will most definitely happen!)</span>


---
layout: two-cols-title
---

::title::

# Self-host - yay or nay?  <span class='small'> (no one-size-fits-all answer)</span>

::left::

### You should probably self-host if...


- You must run on‑prem,  air‑gapped or have other sovereignity constraints <span class='small'>(certification issues, data privacy, ...)</span>
- You need customization 
  - Special networking needs  <span class='small'> (high specialized interconnect for GPU nodes, ... )</span>
  - Tight control over k8s internals  <span class='small'> (custom scheduling, admission controllers, special etcd setup, ...)</span>

And you have:

- Team with ==infra experience== and capacity to
  - Design and maintain a cluster stack
  - Implement and *test* desaster reovery
  - Handle security patches, upgrades

::right::

###  When you probably should not self‑host

- You're already on AWS/Azure/GCP
- You have a small team
- You don't have hard regulatory / locality constraints
- You need a lot of scaling. Quickly.

Typical symptoms:

- "We'll figure out backups later"
- "Networking will be easy, right?"
- "We don't have time for DR tests"

> If you don't have strong reasons *for* self‑hosting,
> your default should probably be managed k8s.


---
layout: section
color: light
---

# From decision to reality

We assume we have ==good reasons to self‑host==  
What does life with a self‑hosted cluster look like?


<StickyNote  color="green-light" textAlign="center" width="180px"  v-drag="[632,157,222,193,5]" v-click>
 
 <br>
 
 ## ..and how can we build it now?

 </StickyNote>

---
layout: default
---

# Life of a self‑hosted cluster

<div style="display: flex; justify-content: space-between; margin-top: 6rem">

<div>

**Day 0** - ==Planning==
- Decide cluster topology
- Find suitable OS & k8s distro
- Tool selection
- Consider delivery pipeline
</div>

<div v-click=1>

**Day 1** - ==Bootstrapping==
- Provision hardware / VMs
- Form initial control plane & worker set
- Install CNI, ingress, cert management, ...
- Set up basic security 
- Deploy initial apps
</div>

<div v-click=2>

**Day 2** - ==Operations== <span class="small">(aka, the grind)</span>
- Regular upgrades (OS, k8s, ...)
- Security updates, incident response
- Capacity changes (scale out/in)
- Backup & DR drills
- Refine observability (metrics/logs/traces)
</div>

</div>

<br>
<br>


<Admonition title="Day 0 is (most) important!" color='emerald-light'  v-click=3>
Strong planning in the beginning helps to avoid pain on Day 2 - this includes proper tooling!
</Admonition>


---
layout: default
---

# Typical example on day 2 - `upgrade k8s`

1. **Plan**
   - Review change impact <span class='small'>(Downtime?)</span>
   - Decide [target version](https://kubernetes.io/docs/tasks/administer-cluster/cluster-upgrade/) & timeline
2. **Update desired state**
   - Update cluster config <span class='small'>(ideally IaC)</span>
3. **Rollout**
   - Control plane upgrades <span class='small'>(highly available? proper failover?)</span>
   - Worker nodes drained, upgraded, returned to service
4. **Verification**
   - Check metrics, logs, error budget
   - Run smoke tests
5. **Fallback**
   - If issues: roll back to previous version <span class='small'>(Proper rollback mechanism in place?)</span>


---
layout: default
---

# The laundry list 🧺
The minimal ==ingredients== for a self-managed cluster:

- Hardware / VMs
- Operating system (OS)
- Kubernetes distro / flavor
- Cluster formation & lifecycle
- Networking (inside & outside)
- Storage
- Backup & recovery
- Security
- Observability


<img src="/img/laundry-list.svg" v-drag="[454,159,448,269]" />

<StickyNote text-align=center color='emerald-light' v-drag="[263,339,227,127,2]" v-click="1">


## OSS ftw!

There's excellent **open-source** tooling for all of this!

</StickyNote>

---
layout: two-cols-title
---

::title::
# OS considerations

::left::

### Container only clusters

- Humans rarely SSH in
- Nodes are treated as ==cattle== (not pets)


**Container‑only clusters**

- Prefer minimal, container‑optimized OS
- Examples
  - Fedora CoreOS
  - Flatcar
  - Talos <span class="small">(fully API driven)</span>
- Benefits:
  - Immutable / declarative <span class="small">(very little config drift)</span>
  - Easier upgrades & reproducibility


::right::


###  Mixed usage clusters
- Prime example: AI clusters
- People run experiments, scripts, fine‑tuning jobs directly on nodes
- More drift, more risk

**Mixed‑use clusters**

- Use what your people know:
  - Ubuntu LTS, Debian, RHEL, …
- Be aware:
  - Higher drift and config entropy
  - Security & stability require more discipline <br>
    <span class="small">(convention rather than strict enforcement)</span>


> If possible, separate “playground” machines from more productive nodes

<!-- 

- What do we need to consider? Not much.. but it depends on your use case
- Ideal: Only containers, no interaction -> Something minimal that is OSS. Fedora Core OS is an excellent choice
- Often: Mixed -> Some people need to access
  - Prominent example: AI clusters. People need to do research, start scripts for fine tuning or “quickly try something”.
  - Requires access to servers. Typically container-optimized OS don’t do so well, often read only.
  - You’ll be fine with any OS, use one that the people know. I’ll admit: We have Ubuntu and it works (not my favorite choice). A better choice perhaps Debian?
  - In the end we need to figure out which fight to fight.
  - Watch out: Things happen (security is harder, scripts have memory leaks, ...)

 -->

---
layout: two-cols-title
---

::title::

# Kubernetes: vanilla vs. distros

::left::

### Vanilla upstream k8s

- Core components as shipped by the project
- Typically bootstrapped with `kubeadm` or similar
- You assemble surrounding tooling
- aka ["the hard way"](https://github.com/kelseyhightower/kubernetes-the-hard-way)
::right::

### Distributions

- Upstream k8s +:
  - Opinionated defaults
  - Installers & upgrade tooling
  - Integrations (monitoring, registry, etc.)
- Wide spectrum  <span class="small">(60+ CNCF certified distros)</span>
  - Lightweight: k3s, k0s, microk8s
  - Full enterprise, e.g. OpenShift

> We’ll aim for something pragmatic in the middle.

::note::
\[1\] https://www.mirantis.com/blog/kubernetes-distributions-which-option-is-best-for-your-organization-/

<!-- 
Vanilla” k8s vs. Distros
- Vanilla
  - "as is"
  - no shortcuts
- Distro
  - Upstream packaged with defaults, tools and support
    - Often their own container registry
  - Often upgrade utilities, easy setup
  - Varying degrees - from heavily integrated to lightweight
  - [show CNCF landscape]
  - If you want, all the versions of the cloud vendors are their own distributions (and heavily integrated at that)
- What makes it different?
  - Often easier workflow for setup, upgrade, patching
  - Great integrations
  - Cool for small deployments on the edge: k3s, micro-k8s
  - The more enterprise, the more heavy

 -->

---
layout: default
---

# Solid pick - `rke2`

**[rke2](https://docs.rke2.io/) - Rancher Kubernetes Engine 2**

- CNCF‑conformant Kubernetes distribution <span class="small">(fully open source)</span>
- Key features:
  - ==Secure== defaults (hardened configuration)
  - ==Simple== install & upgrade
  - Works on VMs and ==bare‑metal==

Why it is good for self‑hosting:

- Easy to get a secure cluster
- Good documentation & ecosystem
- Less boilerplate than rolling everything with bare kubeadm

Alternatives for small setups:

- k3s - lightweight, great for edge / small clusters
- microk8s - simple developer / lab environment, but also OK for PROD


<img src="/img/rke2-logo.png" v-drag="[606,354,294,95]">

::note::
\[1\] rke2 went by the name of "rancher government" in the past


<!--
- Tool of choice: rke2
- Easy, secure by default
- There’s others, for varying degrees of sophistication and use-cases
-->

---
layout: two-cols-title
---

::title::

# Cluster formation - from `nodes` to `cluster`

::left::

<mark class='red'>Problem</mark> Given some nodes, how do we:

- Initially form a cluster?
- Add nodes?
- Upgrade nodes?
- Recover a broken node or control plane?

Common <mark class='green'>solutions</mark>

- Manual setup (good for learning, **bad** for PROD)
- `kubeadm` + automation (Ansible, Terraform, …)
- Distro‑specific installers (e.g. rke2/k3s installers)
- Higher‑level frameworks (Cluster API, Rancher, …)
- Network boot / PXE for large fleets

::right::

<div v-click=1>

For **small to mid‑size** clusters (up to ~20 nodes):

- Use a distro with a clean bootstrap story (e.g. rke2)
- Wrap it with **simple automation**:
  - Ansible, or Terraform + scripts
  - For small clusters even a manual process is OK <span class='small'>(not great tho)</span>
- Keep the process documented, re‑playable and versioned in `git`

<br>

For **larger** clusters consider:
  - PXE / network boot
  - Centralized image management
  - Cluster API or equivalent

<br>

> Don’t over‑engineer provisioning for a 5‑node cluster <br>
> Don’t under‑engineer for a 200‑node fleet

</div>


---
layout: two-cols-title
---

::title:: 

# Let's set up a cluster!

:: left::

As our first step, we'll set up our k8s cluster:
- We assume that the machines are provisioned and an OS is installed
- We can reach them via SSH 

### Target state
- Use `rke2` as our distro
- One master node, five worker nodes <br><span class='small'>(better would be a highly available setup - but we'll have to save some time)</span>

### Steps
1. Set up the ==server== node (also often referred to as "master")
2. Connect five ==agents== (or "worker")
3. Make sure all nodes join properly and are in `Ready` state



::note::
\[1\] [rke2 installation guide](https://docs.rke2.io/install/quickstart)<br>
\[2\] [rke2 high-availability setup](https://docs.rke2.io/install/ha)


---
layout: two-cols-title
---

::title::

# Networking - internal and external

::left::
### Inside the cluster
- **CNI (Container Network Interface)** 
- Pod‑to‑Pod / Pod-to-Node networking
- Services, DNS
- Network policies


**Modern choice: Cilium** 

<img src="/img/cilium.png"  v-drag="[184,263,66,65]">

- Killer features:
  - Transparent encryption between nodes
  - L3/L4/L7 network policies
  - Great observability (Hubble)
- Works well on bare‑metal and cloud VMs

Most important: Pick one and ==avoid switching later==

::right::

### Outside to inside
- Client traffic into the cluster
- External LB → Ingress controller → Services → Pods
- Ingress, TLS termination

<br>

**Ingredients**

- **Ingress controller**
  - NGINX Ingress Controller is <mark class='red'>discontinued</mark>
  - Replace with [Traefik](https://doc.traefik.io/traefik/reference/install-configuration/providers/kubernetes/kubernetes-ingress/)
- **TLS certificates**
  - [cert-manager](https://cert-manager.io/) + ACME (Let’s Encrypt) or internal CA
- **Load balancer**
  - [MetalLB](https://metallb.io/) or Cilium (relatively new)



---
layout: two-cols-title
---

::title:: 

# Step 2 - set up proper networking!

:: left::

### In-cluster: Cilium 

- Adapt the rke2 config file - Cilium is supported out-of-the-box!
  ``` yaml
  # /etc/rancher/rke2/config.yaml
  cni: cilium
  ```

- Restart the rke2 service
  ``` bash
  systemctl restart rke2-server
  ```

- Add encryption via a ==HelmChartConfig==

::right::
### External: Configure Cilium 

- rke2 brings it's own <mark class='green'>Ingress Controller</mark> - **nothing to do here!**
- Need to add support for external IP addresses for ==LoadBalancer== services
- Requires two resources:
  - ==CiliumLoadBalancerIPPool== -> Which IPs are available?
  ``` yaml
  apiVersion: "cilium.io/v2"
  kind: CiliumLoadBalancerIPPool
  metadata:
    name: "blue-pool"
  spec:
    blocks:
    - cidr: "10.0.10.0/24"
  ```
  - ==CiliumL2AnnouncementPolicy== -> Make the IPs known!
- [Simply apply these resources](https://docs.cilium.io/en/latest/network/lb-ipam/) and we're done

 

::note::
\[1\] rke2 [networking](https://docs.rke2.io/networking/basic_network_options) guide<br>
\[2\] Notable alternative for external is [MetalLB](https://metallb.io/)



---
layout: two-cols-title
---

::title::
# Storage: what problem are we solving?

::left::

- Caches / temp data for some services <span class='small'>(not critical)</span>
- <mark class='red'>Actual state</mark>
  - Databases
  - Message queues
  - User‑uploaded data

<mark class='green'>Try to avoid stat in the cluster if possible!</mark>

**Challenges**
- I want `150GB` with fast access now. And replicated.
- Storage is ==high risk, low perceived reward==
  - Everyone expects it to “just work”
  - When it doesn’t, the fallout is huge

We need a solid approach with ==backups== (and restore...)

::right::
<div v-click>

##### Local storage on nodes
- Simply use the disks of the nodes
- Does not scale well, sometimes enough though for small clusters

<div style="margin-bottom: 1.5rem"/>

##### In-cluster replicated storage
- Distributed storage system built from node disks
- Good for general purpose PVs

<div style="margin-bottom: 1.5rem"/>

##### External storage system
- NFS / SAN / NAS solutions
- More "enterprise style" <span class='small'>(probably overkill to set upfor small/medium setups)</span>

<div style="margin-bottom: 1.5rem"/>

##### State out of the cluster
- Especially for DBs it is worth considering <span class='small'>(or a dedicated DB cluster)</span>
- Use object store (S3) wherever possible  <span class='small'>(logging and metrics for instance)</span>


</div>
<!-- 

self-hosted:
- A storage system (local disks, NFS, SAN, Ceph, etc.).
- The matching CSI driver and/or dynamic provisioner.
- A StorageClass that uses that driver.

flow:
- Developer creates a PVC: “I need 10Gi, StorageClass=fast-ssd”.
- Kubernetes asks the CSI driver for fast-ssd to provision a volume.
- The driver creates the volume in your storage system
- When a pod using that PVC is scheduled, the CSI driver:
  - Attaches / mounts the volume on the node.
  - Kubernetes mounts it into the pod.

 -->



---
layout: default
---

<img src="/img/longhorn.png" v-drag="[565,373,332,69]">

# Tool of choice: Longhorn

### CNCF project for ==distributed block storage== on k8s
Provides:
  - A [CSI](https://github.com/container-storage-interface/spec/blob/master/spec.md) driver
  - Replication across nodes
  - Snapshots, backup/restore


<div style="margin-bottom: 1.5rem"/>

How it fits:

- PVC → dynamically provisioned volume
- Longhorn → manages volumes replicated across nodes
- Backup → to external object storage


<div style="margin-bottom: 1.5rem"/>

Why it's pragmatic:

- Easy to install and operate (relative to alternatives)
- Works well for small/medium self‑hosted clusters

---
layout: two-cols-title
---

::title:: 

# Step 3 - storage with Longhorn

:: left::

### Install Longhorn via the helm chart

- Add the repo
  ``` bash
  helm repo add longhorn https://charts.longhorn.io
  ```
- Install the chart
  ``` bash
  helm install longhorn longhorn/longhorn \
    --namespace longhorn-system \
    --create-namespace \
    --version 1.10.1
  ```
- Verify the installation
  ``` bash
  kubectl -n longhorn-system get pod
  ```
- Also verify by checking out the ==Longhorn UI==



<StickyNote  color="amber-light" textAlign="left" width="180px"  v-drag="[621,155,253,232,5]" v-click=1>
 
 ## `PersitentVolumeClaims` now are no longer pending - they will be satisfied right away with a freshly created `PersistentVolume`

 </StickyNote>

::note::
\[1\] [longhorn](https://longhorn.io/docs/1.10.1/deploy/install/install-with-helm/) setup guide

---
layout: two-cols-title
---

::title::

# Backups & recovery ==aka "hope is not a strategy!"==

::left::

### What do we need to backup?

- **etcd**
  - Cluster state
  - API objects (Deployments, Services, etc.)
  - <mark class='red'> Secrets </mark>
- **Persistent volumes**
  - Application data on PVs
- **Databases**
- **Message Queues**


<StickyNote color='emerald-light' v-drag="[51,390,317,80]" text-align=center>

#### We don't have to backup configuration if we stick to CaC / Gitops

</StickyNote>


::right::

### Tools and patterns

- **etcd**
  - Regular snapshots <span class='small'>(rke2 default: every `12h`)</span>
  - Store <mark class='red'>off‑cluster</mark>  <span class='small'>(e.g., object store)</span>
  - Practice restoring to a new control plane

- **PVCs**
  - Essentially a [CSI snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/) + backup pipeline
  - Velero implements this for backups & restores
  - Longhorn also has backup mechanism

- **Databases**
  - ==Use DB‑native backup tools==  <span class='small'>(`pg_dump`, WAL archiving, operator capabilities, ...)</span>
  - Store backups off‑cluster


::note::
\[1\] Example with [Longhorn + Velero](https://longhorn.io/blog/20250902-k8s-backup-solutions-and-longhorn/)

---
layout: two-cols-title
columns: is-7
---

::title::

# Security - a `continuous` concern

::left::

### Security is not a single setup step - ==it's a mindset==

Built-in k8s have functionality to help us here:
- Service accounts & RBAC
- Isolation primitives (namespaces, network policies)
- Pod security controls
- Integrating external identity & secret backends


<StickyNote color="red-light" v-drag="[54,347,436,126,-2]">
<br>

### Security is a `huge` topic (lecture on its own) - this is merely a list to get you started!

</StickyNote>

::right::

### Baseline security checklist

- **Cluster access**: Avoid the `cluster-admin` kubeconfig, ideally OIDC with an *external* identity provider
- **RBAC**: Least privilege for users & service accounts
- **Pod Security**
  - [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) (admission policies)
  - Enforce things like `runAsNonRoot`, no privilege escalation, seccomp, AppArmor
- **Network policies**
  - Default deny, explicitly allow *only*  required traffic
- **Secrets**: External secret stores ([OpenBao](https://openbao.org/) + External Secrets Operator), SOPS
- **Encrypted backups**: Often forgotten / leaked
- **Image supply chain**
  - Image + dependency scanning in CI (Trivy, …)

Use CIS benchmarks as a ==guide==, not as dogma.

::note::
\[1\] Running CIS benchmark with [kube-bench](https://www.cncf.io/blog/2025/04/08/kubernetes-hardening-made-easy-running-cis-benchmarks-with-kube-bench/)

---
layout: two-cols-title
columns: is-7
---

::title::

# Security - beyond the basics

::left::

- **Dependencies:** Image signing (cosign) + SBOMs
- **Strict admission policies**
  - Consider using `Restricted` Pod security standard
  - [Kyverno](https://kyverno.io/) is a great policy engine
- **Runtime Enforcement / full container security lifecycle**
  - For example scanning of running processes inside of containers
  - More on the enterprise side, requires effort to do right (and useful)
  - Tools: Tetragon, NeuVector


<div style="margin-bottom: 1.5rem"/>

### Especially for on-premise clusters
- **Network setup** can become very complex!
  - Proper segmentation (VLAN, routing zones)
  - OOB network setup
  - Firewall settings -> Often done <mark class='red'>manually</mark> by a separate team

<StickyNote color="amber-light" v-drag="[633,180,257,216,2]">
<br>

### Those things bring value - but also require `operational effort`!

Recommendation: Explore as you go, see what fits.

</StickyNote>

---
layout: two-cols-title
---

::title::

# Observability (bonus, but crucial)

::left::

### Metrics
  - Prometheus
  - For multi-tenancy: Thanos, Mimir
  - Good starting point: [kube‑prometheus‑stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) <br><span class='small'>(includes node‑exporter, kube‑state‑metrics)</span>
### Logs
  - Loki 
  - OpenSearch

### Traces
- Tempo
- Jaeger

::right::

### Dashboards
  - Grafana

### Alerts
  - Alertmanager
  - Integrate with Slack, email, PagerDuty, …


<StickyNote v-drag="[550,341,271,152,6]">

<br>

### We can start simple, but we do need *something* here from day 1

</StickyNote>

---
layout: two-cols-title
---


::title::

# Multi‑cluster (bonus) 

<img src="/img/k8s.png"  v-drag="[501,18,55,54]" />

<img src="/img/k8s.png"  v-drag="[563,17,55,54]" />

<img src="/img/k8s.png"  v-drag="[626,16,55,54]" />

<img src="/img/k8s.png"  v-drag="[687,16,55,54]" />


::left::

Often you eventually need ==multiple clusters== 

- Separation by:
  - Environment (`dev` / `stage` / `prod`)
  - Region / site
  - Tenant / business unit
  - stateful vs. non-stateful
    <br><span class='small'>(e.g., separate DB cluster)</span>
  - Central control plane


<div style="margin-bottom: 2rem"/>


### Challenges

- Consistent provisioning ==across clusters==
- Managing shared components and policies 
  <br><span class='small'>(while not repeating ourselves too much)</span>
- Federated observability and identity

::right::

### Tools & patterns

- [Cluster API](https://cluster-api.sigs.k8s.io/) (k8s "native")
- Self-managed control plane tools 
  <span class='small'>(Rancher, Openshift, ...)</span>
- Cluster-managers, notably [Gardener](https://gardener.cloud/)
- GitOps orchestrating multiple clusters "by hand"
- Control-plane managers run control planes as Pods
  <span class='small'>(e.g., Kamaji)</span>


---
layout: section
color: light
---

# GitOps

Making k8s operations and deployments
==boring, repeatable and auditable==

---
layout: two-cols-title
columns: is-5
---

::title::

# Control loops & reconciliation

::left::

### Core k8s idea

- You declare **desired state** <span class='small'>(YAML, manifests, charts, ...)</span>
- ==Controllers== continuously reconcile:
  - Compare desired vs actual
  - Take actions to converge <mark class='red'>actual</mark> → <mark class='green'>desired</mark>
- Example: `Deployment`
  - Replicaset controller watches desired number of pods
  - If the number does not match what we ordered: create a new pod


<div style="margin-bottom: 2rem"/>

### GitOps extends this idea

- Desired state lives in **Git**
- A controller reconciles **cluster state** to what’s in Git


<img src="/img/reconciliation.png" v-drag="[411,128,533,355]"/>

---
layout: default
---

# What is GitOps?

<div style="margin-bottom: 5rem"/>

### Git as `single` source of truth
  - Cluster configuration  <span class='small'>(CNI, ingress, certs, observability)</span>
  - Security policies
  - Application deployments

<div style="margin-bottom: 2rem"/>

### Automatic reconciliation
  - A controller monitors the git repo
  - Applies changes to the cluster
  - Continuously keeps cluster in sync

<div style="margin-bottom: 2rem"/>

<img src="/img/gitops.svg" v-drag="[487,59,337,405]"/>


::note::
\[1\] https://www.gitops.tech/

<!--
Main goals:
- consistent, repeatable deployments
- automated deployments and rollbacks
- Apply devops principles:version contol, ci/cd, infrastructure automation

Why it’s useful:
- Every change is a PR (reviewable, auditable)
- Rollback = `git revert`
- Reproducible environments
- Less “kubectl from my laptop at 2am”


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


What is not GitOps?
- Push from a pipeline when a new commit happens

-->



---
layout: two-cols-title
---

::title::

# Why GitOps? <span v-click=1 style="font-size: 3rem"> - on day 1?</span>

::left::

- Infrastructure / Config ==as Code==
- Manage all infrastructure with ==known tools and workflows==
  <br><span class='small'>(everyone knows git and pull requests)</span>
- Maintain <mark class='green'>consistency</mark> and avoid <mark class='red'>drift</mark>
- Full auditability
- Easy (and reproducible) ==rollbacks==
  <br><span class='small'>(requires some help from workloads though)</span>
- Easier / more controled access management
  <br><span class='small'>(credentials stay in the cluster)</span>

::right::

<div v-click=1>

> “We’ll just add GitOps later.”


<div style="margin-bottom: 2rem"/>


### Problems with that

- accumulate <mark class='red'>snowflake state</mark> in the cluster
- migration to GitOps later:
  - Means reconciling state drift
  - Is painful and time‑consuming

<div style="margin-bottom: 2rem"/>

### Better approach

- ==Start simple== with GitOps:
  - Maybe 1 repo, minimal patterns
- Grow patterns as you need them

</div>


---
layout: two-cols-title
---

::title::

# Without vs with GitOps

::left::


### Without GitOps

- `kubectl apply` from laptops
- Imperative `helm upgrade` commands
- Hotfixes nobody ==documented==
- Hard to ==reproduce== a cluster
- “Works on prod. Don’t touch it”

<div style="margin-bottom: 1.5rem"/>

### With GitOps

- All ==changes via PRs==
- `main` branch defines cluster state
- Reverts roll back changes
- Easy to spin up a new cluster with the same config
- Clear audit trail of *who* changed *what* and *when*


::right::


<div v-click>

### Upgrade Ingress controller without GitOps ...


<div style="margin-bottom: 1rem"/>

1. Engineer edits values file locally
2. Runs `helm upgrade` from laptop
3. Something breaks
4. Nobody remembers exactly what changed <br><span class='small'>(classic scenario: incomplete-outdated Confluence pages)</span>
5. Rollback is manual and stressful


<div style="margin-bottom: 1.5rem"/>

### ... and with GitOps


<div style="margin-bottom: 1rem"/>

1. PR updates ingress Helm chart version in Git
2. GitOps controller rolls out change
3. Observability indicates health
4. If issues:
   - Revert commit
   - Controller rolls back
5. Everything recorded in Git history

</div>

---
layout: two-cols-title
---

::title::

# There's really only two GitOps tools

::left::

<div style="margin-bottom: 5rem"/>


<img src="/img/argocd.png" v-drag="[41,128,141,65]"/>

  - `Application` as the basic packaging unit
  - Supports “app of apps” pattern
  - Great UI and visualization
  - Sophisticated workflows + further integrations <br>
    <span class='small'>(Argo Rollouts, Argo Workflows, ...)</span>

<div style="margin-bottom: 5rem"/>


<img src="/img/flux.png" v-drag="[50,339,98,51]"/>

  - More "k8s native"  <span class='small'>(k8s controllers, native RBAC, ...)</span>
  - Lightweight, easy to integrate into Git workflows
  - No UI, mostly CLI based



::right::
### Both tools ...
- continuously reconcile from Git to cluster
- support Helm, Kustomize, raw manifests
- are configured with CRDs
- are CNCF projects and battle‑tested



<StickyNote v-drag="[558,329,212,125,6]" color='emerald-light' text-align=center>

<br>

### Pick one, don’t try to run both.

</StickyNote>

---
layout: two-cols-title
columns: is-7
---

::title::

# Install ArgoCD and our first `Application`

::left::

- Add the repo
  ``` bash
  helm repo add argo-cd https://argoproj.github.io/argo-helm
  ```
- Install the ArgoCD helm chart
  ``` bash
  helm upgrade --install \
    argocd argo/argo-cd \
    --create-namespace -n argocd 
  ```
- Verify the installation
  ``` bash
  kubectl -n argocd get pod
  ```
- Also verify by checking out the ==ArgoCD UI== (port forward)
  ```bash
    kubectl -n argocd get secret \
      argocd-initial-admin-secret \
      -o jsonpath="{.data.password}" \
      | base64 -d
    ```

::right::

``` yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: first-app
  namespace: argocd
spec:
  project: default
  destination:
    namespace: first-app
    server: https://kubernetes.default.svc
  source:
    targetRevision: main
    path: first-app
    repoURL: https://github.com/...
  syncPolicy:
    automated:s
      enabled: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

::note::
\[1\] ArgoCD helm chart [reference](https://artifacthub.io/packages/helm/argo/argo-cd)

---
layout: two-cols-title
columns: is-5
---

::title::

# Apps-of-apps pattern

::left::

### One parent app to rule them all!

<div style="margin-bottom: 3rem"/>

```yaml {|13}
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
spec:
  project: default
  destination:
    namespace: default
    server: https://kubernetes.default.svc
  source:
    targetRevision: main
    path: apps
    repoURL: https://github.com/...
```

::right::


<img src="/img/app-of-apps.svg" v-drag="[441,176,505,225]">

---
layout: two-cols-title
columns: is-7
---

::title:: 

# Typical GitOps repo structure (simple)

::left::

```
config-repo/
|- apps/
   |- app-of-apps.yaml
   |- infrastructure.yaml
   |- first-app.yaml
      |- ...  

|- infrastructue/
   |- longhorn-values.yaml
      |- ...  

|- first-app/
   |- templates/
      |- deployment.yaml
      |- service.yaml
      |- ...  
   |- values.yaml 
   |- Chart.yaml

|- ...  
```
:: right::

- `apps/`
  - Individual applications

- `infrastructure/`
  - CNI, ingress controller, cert‑manager
  - Longhorn, monitoring, logging, security policies


<StickyNote v-drag="[669,307,184,163,3]" color='emerald-light' text-align='center'>

<br>

#### Good for simple setups with or one repo per environment

</StickyNote>

---
layout: two-cols-title
columns: is-7
---

::title:: 

# Typical GitOps repo structure (multi env)

::left::

```
config-repo/
|- apps/
   |- first-app.yaml

|- charts/
   |- infra/
   |- first-app/

|- common/
   |- first-app.yaml

|- envs/
   |- staging/
      |- first-app.yaml
   |- prod/
      |- first-app.yaml
      
```

:: right::

- `charts/`
  - helm chart definitions to re-use for all envs

- `common/`
  - common settings for all envs  <span class='small'>(e.g. network policies)</span>

- `envs/`
  - environment-specific overrides <span class='small'>(e.g. resource allocation, routing)</span>


<StickyNote v-drag="[669,321,191,188,3]" color='red-light' text-align='center'>

<br>

## NEVER use `branches` for environment promotion!!!11

</StickyNote>

::note::
\[1\] [Seriously, don't use branches](https://codefresh.io/blog/stop-using-branches-deploying-different-gitops-environments/)

---
layout: default
---


# GitOps is not a silver bullet

- Environment ==promotion== not directly covered
  - How to bring config and artifacts from `dev` → `stage` → `prod`
- Does not directly integrate into developer pipeline
  - Can lead to slower developer experiences
    <br><span class='small'>(require extra PR to release rather than simple stage in the pipeline)</span>
  - Automated ==end-to-end e testing== not straightforward
- Custom validation required
  - Preview changes from pipeline not always easy
- Dynamic enviroments are harder to deal with
- Secret management 
  - We cannot simply check in secrets to the repo
  - (Some say, it just highlights the problems..)
  - Typical tools: SOPS, SealedSecrets, ...
- Bootstrapping (and its versioning)
- Non k8s state can be harder to manage
  <br><span class='small'>(e.g., DB schema upgrades)</span>

<div style="margin-bottom: 1rem"/>


---
layout: section
color: light
---

# Summary & decision framework
==quick recap & cheat sheet==


---
layout: default
---

# Simple decision flow

- On a major cloud, no strong constraints? ==managed k8s==
- Must run on‑prem / air‑gapped / sovereign? ==Self‑hosting== is the way to go (it's doable!)
- Is the platform part of your model? ==Self‑hosting== is almost unavoidable
- Team size & skills:
  - If you cannot afford at least some dedicated infra capacity (and regular DR tests) <mark class='red'>think twice</mark>
  - The higher your SLO, the more staff / discipline you'll need (GitOps, o11y, backups and desaster recovery)


<StickyNote color='emerald-light' v-drag="[368,284,226,183,1]" text-align="center">

<br>

### Self‑hosting is `powerful`, but should be a conscious choice.

</StickyNote>


---
layout: two-cols-title
---

::title::

# Cheat sheet for self-hosting stack

::left::

- **Infra & OS**: Fedora CoreOS <span class='small'>(or other container-focused Linux)</span>
- **Kubernetes**: rke2
- **Networking**
  - CNI: Cilium
  - Ingress: NGINX <span class='small'>(not for long)</span> or Traefik
  - TLS: `cert-manager`
- **Storage & backup**
  - Storage: Longhorn
  - Backups:
    - etcd snapshots
    - Velero for PVs
    - ==workload‑native backups== + off‑cluster storage

- **Security**: Start with a basic list, then ==CIS benchmark==

- **Use GitOps from the start**

::right::
