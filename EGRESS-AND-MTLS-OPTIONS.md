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

### What the Chart A rows mean

**Destination expressed as.** All three write the destination as a hostname, which is
why the row reads the same across the chart. What differs is where that hostname
goes.

Cloud Foundry, in the application's metadata and the common file the perimeter list
is built from:

    internet-urls:
      - login.microsoftonline.com
      - sts.windows.net

Option 1, in the application's `FQDNNetworkPolicy`:

    egress:
      - matches:
          - name: login.microsoftonline.com
          - name: sts.windows.net
        ports:
          - { protocol: TCP, port: 443 }

Option 2, in a `ServiceEntry` in the application's namespace:

    hosts: ["login.microsoftonline.com"]
    exportTo: ["."]
    location: MESH_EXTERNAL
    resolution: DNS
    ports:
      - { number: 443, name: https, protocol: TLS }

**Matched on.** This is where they diverge, and it is the row that carries the real
difference.

Cloud Foundry declares by name and enforces by address. The hostname list feeds the
firewall's external dynamic list; the application security group bound to the space
is written as addresses and ports:

    [{ "protocol": "tcp", "destination": "40.126.0.0/18", "ports": "443" }]

Option 1 has the same shape, with the translation automated. GKE resolves each name
in the rule and programs the resulting addresses into the kernel datapath. The
enforcement point sees a destination address, never a name. Over several runs on the
sandbox, `login.microsoftonline.com` resolved to `20.190.135.17`, `20.190.155.131`,
`20.190.155.66` and `40.126.27.128` at different moments, and the rule tracked them.

Option 2 is the only one that enforces on the name. The proxy reads the server name
the client asks for in the TLS handshake and matches the rule against that:

    ClientHello, SNI = login.microsoftonline.com   ->  matches the ServiceEntry
    ClientHello, SNI = something-else.com          ->  no ServiceEntry, rejected

So Cloud Foundry and Option 1 are the same model — declare by name, enforce by
address, with Option 1 removing the manual step of keeping the address list current.
Option 2 is a different model, and the next row is the consequence of that.

### What "separates destinations sharing one address" means

This is the sharpest limitation of the middle column, and it is worth showing rather
than asserting.

**FQDN policy matches on addresses, not names.** GKE resolves the hostnames in each
rule and programs the resulting addresses into the datapath. Nothing inspects the TLS
server name or the HTTP Host header. We measured both halves of this on the sandbox
cluster:

| Case | Result |
|---|---|
| correlator to `login.microsoftonline.com` by its literal IP, with no hostname sent | allowed |
| correlator to an undeclared host by its literal IP | dropped |

So the rule is really "this application may reach these addresses", with the address
list maintained for you by DNS.

**Why that matters.** Large parts of the public internet sit behind shared front
ends. When we resolved `example.com` from the cluster it returned `172.66.147.243`
and `104.20.23.154`, both Cloudflare edge addresses. Those same addresses serve many
thousands of unrelated sites.

So if an application declares a destination behind a shared CDN:

    declared:  supplier-portal.example.com  ->  resolves to  104.20.23.154
    programmed into the datapath:                            104.20.23.154

    a connection to any other site on that same edge address is now permitted,
    because the datapath sees only 104.20.23.154

The policy reads as "this application may reach the supplier portal". What it
enforces is "this application may reach that Cloudflare edge address", which is a
much larger set.

**The same mechanism causes the opposite failure.** When a destination moves to an
address not yet programmed, calls to a permitted host are dropped until the rule
refreshes. And a hostname resolving to more than 50 addresses exceeds the documented
limit, so not all of its addresses are programmed.

**The mesh matches on the TLS server name instead**, which is why Chart A gives
Option 2 a yes on that row. The proxy reads the name the client asked for, so two
destinations on one address are distinguishable.

**How much this affects Énergir is measurable, not theoretical.** There are about 41
distinct external destinations across the estate. Resolving all of them and looking
for addresses that serve more than one would show exactly which pairs cannot be
separated. That is a short exercise and it turns this row from a caveat into a
number.

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
| Resilience against a slow or failing dependency | libraries present but not in use | none in the platform | circuit breaking and outlier ejection per destination |
| Cost | none | none | a proxy container per pod, and every caller must also be in the mesh |

The middle column is already an improvement on what is being replaced: the caller is
identified by workload rather than by address, and both ends must permit the call.
The right-hand column is a further step from there, not a repair of a gap.

Note what the right-hand column costs in Chart A. Every gain here is paid for there,
and only because the two features cannot run together.

### What three of those rows mean

**Path or method control.** NetworkPolicy decides whether a connection may be
opened. It cannot see inside it. So an application permitted to reach another may
call any endpoint on it: if batch is allowed to call correlator, it may call
`GET /api/orders` and equally `DELETE /api/orders/123`, because both are the same
TCP connection to the same port.

`AuthorizationPolicy` in the mesh decides per request, because the proxy reads the
request before forwarding it. A rule can permit batch to call `GET` on `/api/*` and
deny everything else from that caller.

Whether that is worth having depends on whether Énergir wants the platform to
enforce which operations one application may perform on another. Cloud Foundry never
did; the callee's own code decides today, from the Azure AD token in the request.
The mesh would move part of that decision into the platform, where it applies even
if the application's own check is wrong or missing.

**Per-service-pair telemetry, no code change.** Today, answering "which applications
call correlator, how often, and how many of those calls fail" means reading
correlator's own logs and correlating them by hand. Nothing at the platform level
records who called whom.

Because the mesh proxy sits in the path of every call, it records each one: source
workload, destination workload, request count, error rate, and latency percentiles.
That arrives as metrics without any change to application code, and it is the input
to a dependency map of the estate.

The practical value is in incidents and in change. When an application slows down,
per-pair metrics show whether it is the application or something it calls. Before
retiring or changing an application, they show who is actually calling it, rather
than who was declared as a caller. We measured that those two are not the same:
correlator declares two callers in its manifest, and the configuration of other
applications shows three different ones, with no overlap.

**Resilience against a slow or failing dependency.** The failure this addresses is a
familiar one. An application calls another that has become slow rather than down.
Each call waits for a timeout instead of failing fast, the caller's threads fill up
waiting, and the caller stops serving its own traffic. One slow dependency takes out
applications that were healthy.

Two mechanisms in the mesh address it, both configured per destination and neither
requiring application code:

- **Circuit breaking** caps how many connections and pending requests a caller may
  have outstanding to one destination. Past the cap, further calls fail immediately
  rather than queueing. The caller degrades instead of stalling.
- **Outlier ejection** watches for an individual pod returning errors and removes it
  from the pool for a period, then tries it again. A single bad replica stops taking
  a share of traffic without anyone intervening.

The Cloud Foundry column says "libraries present but not in use" because that is what
the configuration showed. Hystrix and Ribbon were in the config layers, but no
application uses `@HystrixCommand` and `feign.hystrix.enabled` was already false. So
nothing was actually breaking circuits. They were removed during conversion, and
removing them lost no behaviour that was in effect.

That makes this row honest in a way worth stating: the mesh would add resilience the
estate does not have today and has not had, rather than restore something the
migration took away.

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
