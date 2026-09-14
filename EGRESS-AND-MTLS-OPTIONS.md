# Egress control and mTLS: the two designs

Google Cloud Support, case 75419399, established that FQDN network policy and Cloud
Service Mesh cannot be combined on a namespace enrolled in the mesh dataplane, in
either sidecar or ambient mode. A namespace not enrolled keeps FQDN policy working
normally. The supported way to do per-application egress allowlisting on a meshed
namespace is the mesh's own controls.

That makes this a choice between two designs, not a set of features to combine.

| | Option 1 | Option 2 |
|---|---|---|
| Mesh dataplane | not enrolled | enrolled, STRICT mTLS |
| Internet egress | `FQDNNetworkPolicy` | `ServiceEntry` under `REGISTRY_ONLY` |
| East-west | NetworkPolicy | NetworkPolicy and `AuthorizationPolicy` |
| Encryption between applications | none added | mTLS |

## Two different questions

The decision touches two separate paths, and conflating them is the usual source of
confusion.

```
      inside the cluster                    |     leaving the cluster
                                            |
  batch          ====mTLS====> correlator   |  correlator --- TLS ---> Cloud SQL, over PSC
  mesh ingress   ====mTLS====> correlator   |  correlator --- TLS ---> Firestore
  correlator     ====mTLS====> sonnette     |  batch      --- TLS ---> login.microsoftonline.com
                                            |  batch      --- TLS ---> Geotab
                                            |
  both ends prove identity with a           |  the far end proves identity; the client
  certificate the mesh issues               |  proves it with a token or a credential
```

**Outbound calls are already TLS.** They terminate at the service being called, and
the client authenticates with an OAuth token, a database credential or a Google
service account. The mesh does not change them and is not intended to. What the two
options differ on outbound is *which destinations an application may reach*, not
whether the call is encrypted.

**Inside the cluster is where mTLS applies.** This is the only place the mesh changes
the transport.

## The problem, stated plainly

These are two independent requirements, and nothing about one improves the other.
A service mesh does nothing for outbound containment. FQDN network policy does
nothing for traffic inside the cluster.

They are coupled only because GKE does not support FQDN network policy on a
namespace enrolled in the mesh. That single incompatibility forces a trade between
two things that have no technical relationship to each other:

    keep FQDN policy  ->  per-application egress enforced in the kernel
                          no mTLS between applications

    enrol in the mesh ->  mTLS between applications
                          egress enforcement moves into a proxy in the pod

If the two were supported together, there would be no decision to make: we would run
FQDN policy for outbound and the mesh for inside, and each would do the job it is
good at. The whole of this document exists because they cannot be combined.

## Chart A: outbound egress

The mesh contributes nothing to this chart. It appears only because enrolling in it
removes the mechanism in the middle column.

| | Cloud Foundry today | Option 1: FQDN policy on, mesh off | Option 2: FQDN policy off, mesh-native egress |
|---|---|---|---|
| Per-application outbound allowlist | yes, application security groups | yes | yes |
| Destination expressed as | hostname | hostname | hostname |
| Matched on | address | resolved address | TLS server name |
| Separates destinations sharing one address | no | no | yes |
| Port control | yes | yes | yes |
| Enforcement point | platform | kernel dataplane | proxy in the application's own pod |
| Is that enforcement a security boundary | yes | yes | no, by Google's and Istio's own statements |
| Scope of the on/off switch | per application | per application | mesh-wide ConfigMap |
| Objects per external destination | one list entry | one policy entry | one `ServiceEntry` per namespace |

## Chart B: inside the cluster, application to application

FQDN policy contributes nothing to this chart. It appears only because it is what has
to be switched off to reach the right-hand column.

| | Cloud Foundry today | Option 1: mesh off, FQDN policy on | Option 2: mesh on, FQDN policy off |
|---|---|---|---|
| Transport between applications | cleartext over the overlay | cleartext on the pod network, inside a VPC Google encrypts by default | mTLS end to end |
| Who may call whom | IP-based application security groups | NetworkPolicy, by namespace and pod label | also `AuthorizationPolicy`, by service account identity |
| Caller identity at the transport | none | none | `cluster.local/ns/<namespace>/sa/<service account>` |
| Caller identity in the application | Azure AD token | Azure AD token | Azure AD token, unchanged |
| Path or method control | no | no | yes |
| Per-service-pair telemetry, no code change | no | no | yes |
| Resilience | none in the platform | none in the platform | circuit breaking and outlier ejection per destination |
| Cost | none | none | a proxy container per pod, and every caller must also be in the mesh |

The middle column is already an improvement on what is being replaced: the caller is
identified by workload rather than by address, and both ends must permit the call.
The right-hand column is a further step from there, not a repair of a gap.

Note what the right-hand column costs in Chart A. Every gain here is paid for there,
and only because the two features cannot run together.

## What Option 2 changes in the policy set

`FQDNNetworkPolicy` is replaced. Nothing else is. Of the seven policy shapes the
chart renders per application, six are unchanged and still required.

| Policy | Option 1 | Option 2 |
|---|---|---|
| default deny | required | required |
| allow DNS | required | required |
| allow callers, ingress | required | required, plus `AuthorizationPolicy` |
| allow gateway, ingress | required | required |
| egress to in-cluster applications | required | required |
| egress to private addresses, `ipBlock` | required | required |
| **egress to the internet** | **`FQDNNetworkPolicy`** | **`ServiceEntry` + `REGISTRY_ONLY`** |

The internet egress NetworkPolicy does not disappear in Option 2, it widens. Public
destinations cannot be written as CIDRs, so the layer 4 rule opens 443 broadly and
the per-host decision moves into the proxy.

## The two rows that decide it

**Outbound enforcement is not a security boundary in Option 2.** Google's own
guidance:

> The routing configuration of the mesh should not be trusted as a security boundary
> because there are various ways in which a workload could bypass the mesh proxies.

Making it one requires the layered architecture Google documents: egress gateways on
dedicated nodes, VPC firewall rules stopping direct egress from workload nodes,
NetworkPolicy permitting workloads to reach only the egress namespace, and
authorization policies on the gateway. That is a larger build than `ServiceEntry`
alone, and it is what Option 2 means if it is to replace what Option 1 enforces
today.

**The outbound switch is mesh-wide.** On a managed control plane,
`outboundTrafficPolicy` is set in the `istio-<release-channel>` ConfigMap in
`istio-system`. One setting for every enrolled namespace, so every application's
`ServiceEntry` must exist before it can be turned on.

## What is settled and what is open

Settled: the two cannot be combined on an enrolled namespace; a non-enrolled
namespace keeps FQDN policy; Option 1 is built and tested on a sandbox cluster, with
declared destinations reachable, undeclared dropped, one application's list not
usable by another, port enforced, and an undeclared caller dropped at the callee.

Open with Google: which of the two they recommend for this requirement; whether the
mesh recommendation means `ServiceEntry` alone or the full egress architecture;
whether per-namespace `REGISTRY_ONLY` is supported; confirmation of the enforcement
semantics we measured; and whether any signal distinguishes enforcing from no longer
enforcing.

## References

| Topic | Document |
|---|---|
| FQDN network policy, syntax and limits | https://cloud.google.com/kubernetes-engine/docs/how-to/fqdn-network-policies |
| Egress control with ServiceEntry and REGISTRY_ONLY | https://istio.io/latest/docs/tasks/traffic-management/egress/egress-control/ |
| `ServiceEntry` reference | https://istio.io/latest/docs/reference/config/networking/service-entry/ |
| `Sidecar` reference | https://istio.io/latest/docs/reference/config/networking/sidecar/ |
| MeshConfig on a managed control plane | https://cloud.google.com/service-mesh/docs/enable-optional-features-managed |
| Egress gateway best practices | https://cloud.google.com/service-mesh/docs/security/egress-gateways-best-practices |
| Encryption in transit, VM to VM within a VPC | https://cloud.google.com/docs/security/encryption-in-transit |
| Feature request: FQDN policy with Cloud Service Mesh | https://issuetracker.google.com/issues/292142566 |
