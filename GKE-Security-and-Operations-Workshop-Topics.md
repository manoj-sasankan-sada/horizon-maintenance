# GKE security and operations: topics for the working session

Prepared in response to Énergir's agenda. One section per topic, in the priority order
Énergir set. Each section states what Cloud Foundry did, what is already implemented on
`appli-gke-npr-01` today, the options with their trade-offs, and a recommendation.

Where something is already decided and running, it is described as fact rather than
proposed again. Where a decision is genuinely open, the options are set out so Énergir can
take it.

Two scope notes, so the boundaries are visible rather than discovered later. Configuration of
Palo Alto network infrastructure and of Cloudflare sits with Énergir, as does configuration
of systems outside Google Cloud, which covers Prisma Access. And application code changes for
compatibility with Firestore, Cloud SQL or Cloud Service Mesh are outside this engagement.
Where a recommendation below implies work in one of those areas, it is marked as a follow-on
rather than planned into the migration.

## Where the platform is today

| | State |
|---|---|
| Cluster | `appli-gke-npr-01`, GKE Autopilot, regional, Dataplane V2 |
| Network policy | Enforcing. Default deny in both directions, per application |
| Internet egress | FQDN network policies, generated per application |
| Internal egress | ipBlock rules by address and port |
| In-cluster egress | Namespace and pod selector rules |
| Inbound | Gateway API, two Gateways, one internal and one DMZ |
| Service mesh | Managed Cloud Service Mesh installed on the cluster. Application namespaces are not in the dataplane, and the generated values carry `mesh.enabled: false`. See 1.4 |
| Secrets | Secret Manager, mounted through the Secret Manager CSI add-on at `/var/secrets` |
| Identity | Workload Identity Federation. No service account keys anywhere in the chain |
| Certificates | Gazmet PKI for internal, Cloudflare origin CA for the DMZ path |
| Namespaces | `ns-<env>-<app>`, one per application per environment |
| Applications | 12 in scope for phase 1, 10 converting with no design change |

---

# Priority 1: Network security and traffic control

## 1.1 The shape of the problem

Cloud Foundry expressed all three kinds of connectivity in one file. The manifest declared
`egress.internet`, `egress.wan`, `ingress.apps` and `routes`, and the platform turned those
declarations into two separate things by two separate mechanisms.

**Application security groups**, for east-west and internal access, maintained by the
platform's own dynamic firewall management.

**A URL allow list at the perimeter**, for internet access, enforced by Prisma Access.

That second mechanism is a pull rather than a push, and the shape matters. Prisma Access
holds a Palo Alto **External Dynamic List** object, which contains no destinations at all:
only a URL, a type and a refresh interval. A single security rule references that object
instead of a list of hosts, and that rule is then never edited. Prisma refetches the URL on
its own schedule and applies whatever it finds.

Behind the URL, `cfar api-plus` composes the answer fresh on every request: it reads the
`internet-urls` metadata field for every application through the Cloud Foundry API, pulls
`envs/common/egress_urls/meetme.yml` from the `cfar-bosh-environment` repository, merges the
two, and returns plaintext hostnames with no scheme and no ports. A developer adds a URL to
a manifest, `cfar cli` writes it into the application's metadata, and the next fetch picks
it up.

Three properties of that model are what has to be replaced:

**The declaration lived with the application.** One metadata field per application, no
second place to look.

**A global list covered what every application needs.** Per-application declarations stayed
short because the common case was declared once, centrally: buildpack staging domains,
technical integrations such as `energir.azurecr.io`, and commonly needed endpoints such as
Azure Graph and Entra ID.

**The firewall rule was static while the list was dynamic.** The network team created the
EDL object and one rule once. After that a declaration reached the perimeter without a
change request, because the rule never changed, only the contents of the list it pointed at.

**One thing that model did not provide is per-application separation at the perimeter.**
Prisma receives a single flat union across every application. Per-application enforcement in
Cloud Foundry came from the application security groups, a separate mechanism. That is worth
stating because it sets the bar: a single GKE rule carrying the union is not a regression at
the perimeter, and the control that has to be reproduced is the one the security groups
provided.

Kubernetes provides none of this. It provides a policy object an application team can write,
and nothing that reaches outside the cluster.

## 1.2 Outbound internet access

### What is implemented today

Every application carries a `FQDNNetworkPolicy` listing the public hostnames it may reach,
plus a default-deny egress policy. The destination list is derived from two sources, because
neither alone is complete: the Cloud Foundry manifest's `egress.internet` block, and every
URL in the application's own Spring configuration layers. The second source matters more
than expected: across the estate the configuration layers name roughly 270 destinations in
85 application folders that no manifest ever listed, because Cloud Foundry never required
them to be declared.

The generation step fails closed. A destination that cannot be resolved to a rule stops
generation and names the application, rather than emitting a placeholder that would render a
policy matching nothing.

Separately, and independently of any policy: `vpcs-npr` has no Cloud NAT. A private node has
no direct route to the public internet, so outbound internet traffic leaves through the
corporate perimeter. Permission and reachability are two different things, and both have to
be true.

At the perimeter, a firewall rule now exists from the GKE pod subnet to outside, for NPR, PRD
and DR, carrying a **static copy** of the Cloud Foundry URL list. That is the right way to
start, and it is also the thing that will drift: in Cloud Foundry the list was regenerated
from application metadata, and a copy is regenerated by nobody.

### How the list will be validated

The copied list is a starting point in both directions. It carries entries no pod needs,
because Cloud Foundry staged the application inside the runtime container and a running
application therefore needed the buildpack's own download domains. In GKE the pod stages
nothing: the image is built beforehand and pulled by the node, so those belong to the build
path. And it can be missing entries, because Cloud Foundry never required a destination
reached only from a configuration layer to be declared at all.

Rather than propose a trimmed list from analysis, it will be produced from behaviour. The
twelve applications run through CI and CD across dev, QA and UAT during the migration, under
an enforcing default-deny policy, and anything missing surfaces there. A destination that is
present but unnecessary is visible the same way, from the generated per-application lists.

The output of that exercise is the per-application destination list, accurate per
environment, which is the input whichever mechanism Énergir chooses below.

### The constraint that shapes this topic

Google documents that FQDN network policies and Cloud Service Mesh cannot be used together.
The limitation is stated plainly: "Cloud Service Mesh is not supported."

That has a direct consequence for the target architecture. Today the policy objects for the
mesh exist but the application namespaces are not in the dataplane, so FQDN policies are
enforcing and working. The moment a namespace joins the mesh dataplane, FQDN-based internet
egress control for that namespace is no longer supported, and internet egress has to be
expressed some other way.

So the mesh decision and the internet egress decision are one decision, not two. That is the
single most useful thing to settle in this session.

Other documented limits worth knowing before committing to FQDN policies:

- wildcards match one label only, so `*.company.com` matches `api.company.com` but not `eu.api.company.com`
- at most 50 addresses per FQDN, and 100 per hostname
- the cluster must use kube-dns or Cloud DNS; custom CoreDNS is not supported
- no Layer 7 enforcement, so the path of a URL is not considered
- a ClusterIP or headless Service cannot be an FQDN egress destination
- inter-node transparent encryption is not supported alongside it

### Options

| | Approach | Pros | Cons |
|---|---|---|---|
| A | FQDN network policies, as implemented today | Closest to the Cloud Foundry model: a declared hostname list per application. Native, no extra infrastructure, no cost. Already working and already generated | Incompatible with Cloud Service Mesh. Wildcard and address-count limits. No Layer 7. Shared IP addresses mean allowing one host can allow others behind the same address |
| B | Cloud NAT with a reserved egress address, and perimeter rules against that address | Restores a stable, attributable source address. The perimeter keeps its existing enforcement role and tooling | All applications share one identity at the perimeter, so per-application control is lost unless combined with A or C. Requires a NAT gateway that does not exist today |
| C | Secure Web Proxy, a managed forward proxy with URL policies and optional TLS inspection | Layer 7 control, per-URL rather than per-host. Central policy and logging. TLS inspection possible where it is acceptable. Compatible with a service mesh | New component to run and fund. Applications must be proxy-aware or traffic routed to it. TLS inspection is a significant policy decision in its own right |
| D | Istio egress gateway with `ServiceEntry`, once the mesh is adopted | Consistent with a mesh-based design. Gives outbound traffic a workload identity. Keeps one policy language for inbound and outbound | Only meaningful if the mesh is adopted. Another Envoy tier to operate. Does not by itself create an internet path |
| E | Regenerate the perimeter URL list from the same source that generates the policies, plus a GKE equivalent of `meetme.yml` | Preserves the Cloud Foundry operating model exactly, including Prisma Access and the perimeter team's workflow. Application teams keep declaring in one place. The declaration is already resolved per application per environment, so the list is a publishing step rather than new analysis. More complete than the Cloud Foundry list, because configuration layers are scanned as well as manifests | Requires a job that publishes the list and a defined way for Prisma Access to consume it. The loop is rebuilt by us rather than provided by the platform. A global common list has to be created and owned |

### Recommendation

**A is the position, and it is implemented.** Per-application FQDN policy at the pod is the
successor to the Cloud Foundry application security groups, which is where per-application
egress control actually lived. It is generated from the same declarations Cloud Foundry
used, reapplied on every deploy, and it fails the build rather than emitting a rule that
matches nothing. This is the control that must not be given up, and it is why the mesh is
not enrolled, for the reason in the section above.

**The perimeter stays as it is for the migration.** The static rule DO has built works
today. Its weakness is drift, not exposure: a new destination needs a change request where
Cloud Foundry needed none. That is an operational cost for developers rather than a security
gap, and it is acceptable precisely because the precise layer is in the cluster. A stale
union list at the perimeter cannot let one application reach another's destination, because
the pod policy stops it first.

**E is the natural follow-on, and it is Énergir's to take.** Reproducing the Cloud Foundry
mechanism means serving an endpoint that Prisma's EDL fetches, composed from the live
cluster policies plus a GKE equivalent of `meetme.yml`. The GKE side of that is small: the
per-application lists already exist as objects in the cluster. The surrounding work is not,
and none of it is a migration activity: configuring the EDL object and the rule that
references it, deciding how Prisma reaches a GCP-hosted URL, agreeing who owns the common
list, and proving it alongside the Cloud Foundry path while both platforms run in parallel.

**Scope.** Configuration of Palo Alto network infrastructure sits with Énergir, and so does
configuration of systems outside Google Cloud. The perimeter design, the EDL object and the
firewall rules are therefore Énergir's. What comes from this engagement is the accurate
per-application destination list and the in-cluster enforcement that matches it.

### Operational and security implications

A destination blocked by network policy is dropped, not refused. The symptom is a client-side
hang for the full timeout with nothing in any log. This is the single most important
operational fact about the model, because it makes an incomplete destination list look like
an application fault. Two mitigations are already in place: generation fails closed rather
than emitting a placeholder, and the destination list is derived from configuration as well
as from the manifest.

## 1.3 Outbound access to internal services

### What is implemented today

Internal destinations are expressed as ipBlock rules with an explicit address and port. The
address is required, because a network policy cannot match a hostname for a non-HTTP
protocol. Where a destination moves in the migration, the new address, hostname and database
name are recorded per application and per environment and the connection string is rewritten
to match.

Three internal paths are in use:

| Destination | Path |
|---|---|
| Cloud SQL for SQL Server | Private Service Connect, a forwarding rule in `vpcs-npr`, reached by a corporate DNS name |
| Firestore and other Google APIs | Private Google Access, `restricted.googleapis.com`, `199.36.153.4/30` |
| On-premises and other corporate services | Through the corporate perimeter, with pod addresses preserved |

Two properties of these are worth raising in the session because they constrain design:

Private Service Connect is not transitive across VPC peering. A consumer reaches a service
attachment only from the VPC where the endpoint lives.

The Private Google Access rule is coarse. `199.36.153.4/30` serves every Google API, so
allowing Firestore by address also allows every other Google service. Narrowing that is a
VPC Service Controls decision, not a network policy one.

### Options

| | Approach | Pros | Cons |
|---|---|---|---|
| A | ipBlock rules per destination, as implemented today | Precise, protocol-agnostic, works for SQL and message brokers. No dependency on DNS behaviour | Requires an address per destination, and the address has to be maintained when it changes |
| B | Private Service Connect for every managed dependency | Private addressing, no peering, producer never sees the consumer address. Stable endpoint in the consumer VPC | One endpoint per service. Not transitive, so a hub-and-spoke topology needs care |
| C | VPC Service Controls perimeter around the Google APIs | Turns the coarse `restricted` VIP into real per-service control. Protects against exfiltration to a personal project | A significant governance exercise. Perimeter breaks are a new class of incident to operate |

### Recommendation

A and B are already the design and should continue. C is the item worth putting on the
roadmap: without it, "Firestore is reached privately" is true at the network layer and
unenforced at the service layer, and that gap is easier to close before the estate grows than
after.

## 1.4 Application to application communication

### What is implemented today

Two layers, generated from the same declaration so they cannot disagree.

**Layer 4.** The Cloud Foundry `ingress.apps` block becomes an ingress network policy
naming the caller's namespace and pod. Default deny means anything not named is dropped.

**Layer 7 identity.** The same declaration becomes an Istio `AuthorizationPolicy` naming
the caller as `cluster.local/ns/<namespace>/sa/<service account>`. Once any ALLOW policy
selects a workload, everything not listed is denied.

The address side is now unconditional and separate from the permission side. Every
application gets a ClusterIP Service, so `<app>.<namespace>.svc.cluster.local` always
resolves. Whether anyone may call it is the allowed-callers list. In Cloud Foundry those two
were the same thing: declaring a container-to-container route was what made an application
callable.

The applications also authenticate each other above the transport, with Azure AD client
credentials tokens validated against the configured issuer. That is unchanged by the
migration and is independent of both layers above.

### Options

| | Approach | Pros | Cons |
|---|---|---|---|
| A | Network policy only, plus the application's own token validation | Simplest to operate. No mesh to run or upgrade. Compatible with FQDN egress policies. Already a material improvement on the Cloud Foundry posture, which was IP-based security groups over a cleartext overlay | No transport encryption inside the cluster beyond VPC-level encryption. Authorisation is by namespace and pod label, not by workload identity. No circuit breaking |
| B | Network policy plus Cloud Service Mesh with mTLS `STRICT` | Mutual authentication and encryption on every in-cluster call. Authorisation by Kubernetes service account rather than by label. Circuit breaking and outlier ejection as a platform capability | Every caller must also be in the mesh. Rules out FQDN egress policies. An Envoy tier to operate and upgrade. Ingress must go through the mesh ingress gateway for traffic to carry identity |
| C | Mesh with mTLS `PERMISSIVE` as an end state | Appears to allow a gradual migration | Not recommended. Because the ALLOW policy matches on caller identity, a cleartext caller has no identity and is denied at Layer 7 anyway. It keeps the full operating cost of the mesh while removing the transport guarantee |

### Recommendation

**A is the position: Cloud Service Mesh deployed, applications not enrolled in the
dataplane.** The mesh is installed and running on `appli-gke-npr-01`, which is what the
engagement contracted for. What is not done is labelling application namespaces to join the
dataplane, and the generated values carry `mesh.enabled: false`, so the policy objects are
not rendered either.

**The deciding constraint is the FQDN incompatibility, not a security preference.** Google
documents that FQDN network policies cannot be used with Cloud Service Mesh. Per-application
egress control is a Cloud Foundry capability, delivered there by the application security
groups. In-cluster transport encryption is not, because Cloud Foundry carried that traffic in
cleartext over its overlay. Declining the mesh therefore cannot cause a regression against the
Cloud Foundry posture, while declining FQDN policy would. With no compliance requirement for
in-cluster encryption, the choice follows from that alone.

**The supporting case is that A is already better than Cloud Foundry on this axis.**
NetworkPolicy selects on namespace and pod label, and a container cannot relabel itself,
where a compromised Cloud Foundry container could have held an allowed IP address. The
applications also continue to authenticate each other with Azure AD tokens, which the mesh
would not have replaced.

**Avoid C as an end state**, for the reason in the table.

**This is reversible and nothing is written off.** The chart retains `mesh-policy.yaml`,
`destinationrule.yaml`, `virtualservice.yaml` and a `routing.mode` switch. If Énergir later
wants mTLS, the path is to label namespaces, route inbound traffic through the mesh ingress
gateway so it carries workload identity, and move egress control onto `ServiceEntry` plus an
egress gateway, which is the mesh's own equivalent of FQDN policy. Those three have to happen
in the same window, because doing any one alone breaks traffic. Application changes required
for mesh compatibility sit outside this engagement.

### Operational and security implications

If the mesh is adopted, three things change at once and should be planned as one change:
every caller namespace joins the dataplane in the same window, inbound traffic moves from a
direct route to the mesh ingress gateway so that it carries workload identity, and internet
egress moves off FQDN policies. Doing any one of those alone breaks traffic.

---

# Priority 2: Identity, secrets, certificates and RBAC

## 2.1 Secrets management

### What is implemented today

Secret Manager is the store. The GKE Secret Manager CSI add-on mounts each secret as a file
under `/var/secrets`, and Spring reads that directory as a configuration tree, so one
property is one file. The pod authenticates to Secret Manager as its own Kubernetes service
account through Workload Identity Federation. No service account key exists anywhere in the
chain.

Provisioning is split deliberately. Terraform creates the secret containers and the IAM
grants; the values are entered through the console. That keeps credential values out of
Terraform state, out of the repository and out of shell history.

Cloud Foundry had none of this: binding a service injected credentials into `VCAP_SERVICES`
at push time and the application never saw a secret store.

Two constraints found on this cluster and worth knowing:

The add-on provides the CSI driver and `SecretProviderClass` only. It does not synchronise
to a Kubernetes Secret object, which matters because the Gateway listener requires a real
Kubernetes Secret for its certificate.

The `SecretProviderClass` object is applied per namespace, outside the application chart.
A deploy into a namespace where it has not been applied produces a pod whose secret files are
simply absent.

### Options for provisioning and rotation

| | Approach | Pros | Cons |
|---|---|---|---|
| A | Terraform for containers and IAM, values entered in the console, as today | Values never touch state, a repository or a terminal history. Clear separation between who may create a secret and who may read it | Manual step per secret. Not self-service for application teams |
| B | Add automatic rotation in the add-on, so updated versions are pushed to running pods | Removes the redeploy step when a value changes. Native, no extra component | Requires a recent GKE version. The application must tolerate the file changing underneath it, which Spring config trees do not reload by default |
| C | External Secrets Operator, syncing Secret Manager into Kubernetes Secrets | Produces real Kubernetes Secret objects, which the Gateway and other consumers need. Mature and widely used | A third-party controller to run and patch. Secret values are then also at rest in etcd |
| D | Secret Manager's own synchronisation feature | Native rather than third-party, and covers the same need as C | Newer capability, and the same etcd consideration applies |

### Recommendation

Keep A as the provisioning model, because the separation it gives is the point and it is
cheap. Evaluate B for rotation, with the caveat that a rotated file only takes effect when
the application re-reads it, so for Spring config trees the practical unit of rotation
remains a rolling restart. Reach for C or D only for the consumers that genuinely need a
Kubernetes Secret object, which today is the Gateway certificate and nothing else.

The level of automation worth aiming for: secret containers and IAM grants fully in
Terraform, secret values manual and audited, consumption fully automatic, rotation scheduled
with a named owner rather than event-driven.

## 2.2 Certificate management

### What is implemented today

Three certificates at three hops. The internal Gateway listener serves a Gazmet PKI
wildcard, issued against a certificate signing request so the private key is generated where
it is used and never transferred. The DMZ Gateway serves a Cloudflare origin certificate,
because the Cloudflare zone is Full (strict). The mesh ingress gateway serves its own
certificate, and workload certificates inside the mesh are issued and rotated by the managed
control plane with no operator involvement at all.

Material is held in Secret Manager. The Gateway consumes a Kubernetes Secret, so the current
rotation procedure is a manual pull from Secret Manager and a `kubectl create secret tls`,
four times a year per certificate, since Énergir rotates quarterly.

### Options

| | Approach | Pros | Cons |
|---|---|---|---|
| A | Manual load from Secret Manager, as today | Works now. No new component. The CSR flow keeps the private key with Énergir | Four scheduled manual tasks a year per certificate, each one a diarised expiry risk |
| B | cert-manager with an issuer that drives Microsoft Certificate Services | Issues and renews directly into the Kubernetes Secret the Gateway wants. Removes both the manual load and the CSR round trip. Certificate lifecycle becomes declarative and lives in git | Depends on the internal CA exposing a protocol cert-manager can drive, which is the open question. A controller to run |
| C | Google Certificate Manager | Native, integrates with the load balancer, supports Google-managed and self-managed certificates | Google-managed certificates require public domain validation, which does not apply to a private corporate zone. For a private CA certificate it becomes another place to upload the same material |
| D | Secret Manager synchronisation into the Kubernetes Secret | Removes the manual copy without introducing certificate issuance | Solves half the problem. Renewal is still a CSR round trip |

### Recommendation

B is the target if the internal CA can support it, because it is the only option that makes
the whole lifecycle declarative rather than scheduled. The question to settle is whether
Microsoft Certificate Services exposes an ACME or comparable endpoint that cert-manager can
drive. If it cannot, D is a worthwhile intermediate step, and A stays as the operating
procedure with a named owner and a diarised date per certificate.

Workload certificates inside the mesh are already fully automated and need no decision.

## 2.3 RBAC

### What is implemented today

Cluster administration is held by the platform team. One practical constraint has already
surfaced: creating a Kubernetes `Role` requires `roles/container.admin`, because minting
roles is privilege escalation inside the cluster, and the Editor role is deliberately held
back from RBAC object creation. That is the correct behaviour and it is worth stating
explicitly, because it is the first thing an application team hits.

The deploy path uses no human identity at all. The Jenkins agent authenticates with an Azure
managed identity, federates into GCP, and impersonates a deploy service account. Pods
authenticate as their own Kubernetes service account. No long-lived credential is stored.

### Options

| | Approach | Pros | Cons |
|---|---|---|---|
| A | Namespace-scoped `RoleBinding` per application team, bound to a Google Group | Least privilege by construction. Membership managed where identity is already managed, not in Kubernetes | One binding per team per namespace to create and maintain. Needs the namespace convention to be stable |
| B | Cluster-wide roles per function, with namespace selectors where supported | Fewer objects, simpler to reason about | Blast radius is the cluster. A mistake is estate-wide |
| C | GCP IAM only, with no Kubernetes RBAC beyond the defaults | Single place to manage access | GCP IAM roles are cluster-scoped for GKE, so it cannot express "this team, this namespace" |

### Recommended separation

| Role | Scope | Typical permissions |
|---|---|---|
| Platform team | Cluster | Full administration. Gateways, ComputeClasses, CRDs, RBAC objects, namespace creation |
| Application team | Their own namespaces | Read workloads, read logs, port-forward, read events. No write in production; deploys go through the pipeline |
| Operational support | All application namespaces | Read-only across workloads, events and logs. Restart a deployment where the on-call model requires it |
| Deploy pipeline | Their own namespaces | Write, through a service account impersonated by workload identity federation, never a human |
| Security and audit | Cluster | Read-only, including policy objects |

### Recommendation

A, bound to Google Groups, with the four functional roles above. The two rules that make it
hold: no human identity has write access in production, and read access is broad while write
access is narrow, because most support work is reading.

---

# Priority 2: Namespaces, environment segregation and governance

## What is implemented today

One namespace per application per environment, named `ns-<env>-<app>`, for example
`ns-qa-correlator`. Environments are separated at the cluster level rather than the namespace
level: `appli-gke-npr-01` carries dev, QA and UAT, and a separate production cluster is
planned.

The namespace is load-bearing in more places than it looks:

- it is the network policy boundary, and cross-namespace calls must be declared at both ends
- it carries the mesh dataplane label, so mesh membership is decided per namespace
- it carries the `gateway-access` label that permits a route to attach to a shared Gateway
- it is part of the workload identity principal, so renaming a namespace invalidates the IAM grant that lets a pod read its secrets

That last point is the one that catches people, because the symptom is a permission error at
pod start that reads like a missing secret rather than a stale grant.

## Options for environment separation

| | Approach | Pros | Cons |
|---|---|---|---|
| A | Cluster per tier, namespace per application per environment, as implemented | Production is physically separate, which is the strongest boundary and the easiest to evidence to an auditor. Non-production shares infrastructure and cost. One upgrade to validate before production sees it | Non-production environments share a control plane and node pool, so a noisy neighbour in dev can affect QA. Cluster count grows with tiers, not with applications |
| B | One cluster for everything, namespace per environment per application | Fewest clusters to operate and patch. Lowest fixed cost | Production and development share a control plane. A cluster-wide misconfiguration or a bad upgrade reaches production. Harder to evidence separation |
| C | Cluster per environment | Strongest isolation between every environment | Four clusters to patch, four sets of shared infrastructure, four certificates. Cost and operational load scale with environments |
| D | Project per environment as well as cluster per environment | Adds an IAM and quota boundary to the network boundary. Cleanest billing attribution | The most infrastructure to build and keep aligned. Cross-project networking to design |

## Recommendation

**A is the right model for this estate and should be kept.** Twelve applications over four
environments is not a scale that justifies C or D, and the production boundary that matters
most is already there. If Énergir later requires a harder boundary between UAT and the lower
environments, the step from A to C is additive and does not invalidate the namespace
convention.

**Governance that should go with it:**

The namespace name is a contract, not a label. It appears in IAM grants, network policies,
mesh principals and DNS names, so it should be created by the platform team from a fixed
pattern and never edited.

Namespace creation should carry its labels at creation time. `kubectl apply` of a namespace
manifest that omits a label strips it, and both the mesh dataplane label and the
`gateway-access` label fail silently when absent: no error, just traffic that does not work.

RBAC and the namespace boundary should be the same boundary. A team that owns an application
owns its namespaces in every environment, with write in the lower environments and read in
production.

Resource governance belongs at the namespace too. Autopilot bills on requested CPU and
memory, so a `ResourceQuota` per namespace is the practical cost control, and a
`LimitRange` prevents a workload deploying with no request at all.

---

# Priority 3: Additional security capabilities

## 3.1 Container image vulnerability scanning

| | Approach | Pros | Cons |
|---|---|---|---|
| A | Artifact Analysis on Artifact Registry | Native. Scans on push and continuously re-scans as new vulnerabilities are published. No pipeline change | Findings are advisory. Nothing stops a vulnerable image deploying |
| B | A, plus Binary Authorization | Turns a finding into a gate. Only images meeting an attestation policy may run. Dry-run mode lets the policy be proven before it blocks | A new failure mode in the deploy path. Break-glass has to be designed before it is needed |
| C | A third-party scanner in the pipeline | Consistent with tooling Énergir may already run elsewhere | Another product, another integration, overlapping findings |

**Recommendation.** A now, because it requires no pipeline change and the images are already
built by Cloud Build into Artifact Registry. B as the next step, run in dry-run until the
policy is proven. The natural attestation point is the existing CI pipeline, which already
authenticates with workload identity federation and could attest after the scan and the
SonarQube stage.

## 3.2 Runtime workload protection

| | Approach | Pros | Cons |
|---|---|---|---|
| A | GKE Security Posture, workload configuration and vulnerability scanning | Included, low effort, surfaces misconfiguration against Kubernetes and GKE hardening guidance | Detection only |
| B | Container Threat Detection in Security Command Center Premium | Runtime detection of added binaries, reverse shells, library loading. Managed, no agent to run | Requires the Premium tier. Findings need an owner and a response path or they accumulate |
| C | A third-party runtime agent | Deeper policy and enforcement options | Autopilot restricts privileged workloads, so agent support must be confirmed for Autopilot specifically |

**Recommendation.** A immediately, since it is a setting rather than a project. B if Security
Command Center Premium is already held or is being considered, because it is the option with
no agent to operate on Autopilot. Under C, confirm Autopilot support before evaluating any
product, since the restricted workload profile rules out several.

## 3.3 Hardening available in Autopilot

Autopilot already applies most of the CIS GKE Benchmark by construction: nodes are managed
and not directly accessible, workloads run non-root by default, privileged containers and
host namespaces are refused, and the node image and kernel are maintained by Google. This is
worth stating in the session because the hardening questions that apply to a Standard cluster
largely do not apply here, and effort is better spent on the workload profile than on the
node.

What remains in Énergir's hands at the workload level is already set in the chart: non-root
with an explicit UID, no privilege escalation, and a read-only root filesystem where the
application tolerates it.

**GKE Sandbox** is supported on Autopilot from GKE 1.27.4-gke.800 onward, and needs no node
pool preparation: a pod requests it with `runtimeClassName: gvisor`. It places a user-space
kernel between the container and the host, which is the right control for genuinely untrusted
code.

| Pros | Cons |
|---|---|
| Strong isolation for untrusted or third-party code. No node pool work on Autopilot | A syscall interception cost, which matters for I/O-heavy workloads. Some syscalls are unimplemented. Metadata access is blocked from a sandboxed pod, which breaks workload identity for that pod |

**Recommendation.** Not for this estate. These are twelve first-party Spring Boot
applications built from Énergir's own source by Énergir's own pipeline, and they use workload
identity to reach Secret Manager, which a sandboxed pod cannot do. GKE Sandbox is the right
answer for running code Énergir did not write. It is worth knowing it is available per pod
with no cluster change, so the decision can be taken later for a specific workload rather
than now for the platform.

---

# What we need from Énergir to close these topics

| # | Question | Blocks |
|---|---|---|
| 1 | Confirmed: in-cluster transport encryption is not a compliance requirement, and the goal is parity with the Cloud Foundry posture or better. This settles the mesh question | Closed |
| 2 | Does Énergir want the Cloud Foundry mechanism reproduced for GKE, an endpoint that Prisma's EDL fetches, or is a maintained static rule acceptable as the end state? | Whether follow-on work is scoped. Not required for the migration |
| 2b | Who owns the GKE equivalent of `meetme.yml`, the global list of domains every application needs? | The common half of the perimeter list, under either option |
| 2c | Read access to `cfar-bosh-environment`, or to `envs/common/egress_urls/` | Confirming which entries were per-application and which were common |
| 3 | Is TLS inspection of outbound application traffic acceptable? | Whether Secure Web Proxy is a candidate |
| 4 | Does Microsoft Certificate Services expose an ACME or comparable endpoint? | Whether certificate renewal can be automated |
| 5 | Is Security Command Center Premium held or planned? | The runtime protection option |
| 6 | Which teams should hold which of the four RBAC roles, and are Google Groups available for them? | The RBAC bindings |
| 7 | Is a harder boundary required between UAT and the lower environments? | Whether the cluster-per-tier model stays as it is |

---

# References

- [Control Pod egress traffic using FQDN network policies](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/fqdn-network-policies)
- [About GKE network policy enforcement](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/network-policy)
- [Using GKE Dataplane V2](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/dataplane-v2)
- [Gateway API in GKE](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/gateway-api)
- [Secret Manager add-on for GKE](https://docs.cloud.google.com/secret-manager/docs/secret-manager-managed-csi-component)
- [Harden workload isolation with GKE Sandbox](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/sandbox-pods)
- [GKE Sandbox concepts](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/sandbox-pods)
- [Secure Web Proxy policies overview](https://docs.cloud.google.com/secure-web-proxy/docs/policies-overview)
- [Secure Web Proxy TLS inspection overview](https://docs.cloud.google.com/secure-web-proxy/docs/tls-inspection-overview)
