# Two supported designs for egress control

Google Cloud Support, case 75419399, established that FQDN network policy and Cloud
Service Mesh cannot be combined on a namespace enrolled in the dataplane, in either
sidecar or ambient mode, and that the supported way to do per-application egress
allowlisting on a meshed namespace is the mesh's own controls.

That leaves two designs.

| | **Option 1** | **Option 2** |
|---|---|---|
| Mesh dataplane | not enrolled | enrolled, STRICT mTLS |
| Internet egress | `FQDNNetworkPolicy` | `ServiceEntry` under `REGISTRY_ONLY` |
| East-west | NetworkPolicy | NetworkPolicy and `AuthorizationPolicy` |
| Encryption between applications | none | mTLS |

## What Option 2 replaces, and what it does not

`FQDNNetworkPolicy` is replaced. Nothing else is. Of the seven policy shapes the
chart renders per application, six are unchanged and continue to be required.

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

## The comparison

| | Cloud Foundry today | Option 1: FQDN policy, no mesh | Option 2: mesh, STRICT mTLS, mesh-native egress |
|---|---|---|---|
| **Outbound** | | | |
| Per-application outbound allowlist | yes, application security groups | yes | yes |
| Destination expressed as | hostname | hostname | hostname |
| Matched on | address | resolved address | TLS server name |
| Separates destinations sharing one address | no | no | yes |
| Port control | yes | yes | yes |
| Enforcement point | platform | kernel dataplane | proxy in the application's own pod |
| Is that enforcement a security boundary | yes | yes | **no, by Google's and Istio's own statements** |
| Scope of the on/off switch | per application | per application | **mesh-wide ConfigMap** |
| **Inbound** | | | |
| Per-caller control | yes, `ingress.apps` | yes | yes |
| Caller identified by | platform identity | namespace and pod labels | workload certificate |
| Port control | yes | yes | yes |
| Path or method control | no | no | yes |
| **In transit** | | | |
| Encryption between applications | no | no | yes |
| Proof of which workload called | no | no | yes |
| **Operations** | | | |
| Per-service-pair metrics, no code change | no | no | yes |
| Retries, timeouts, circuit breaking in the platform | no | no | yes |
| Cost per pod | n/a | none | a proxy container, billed on Autopilot |
| Pod start depends on a control plane | no | no | yes |
| Objects per external destination | one list entry | one policy entry | one `ServiceEntry` per namespace |
| Skills to operate | CF platform team | Kubernetes NetworkPolicy | Kubernetes and Istio |

## The two rows that decide it

**Enforcement is not a security boundary in Option 2.** Google's own guidance is
explicit:

> The routing configuration of the mesh should not be trusted as a security
> boundary because there are various ways in which a workload could bypass the mesh
> proxies. The configuration of the outbound listeners in sidecar proxies should
> not, on their own, be considered as security controls.

Istio states the same in its egress task. Making Option 2 an actual control requires
the layered architecture Google documents: egress gateways on dedicated nodes, VPC
firewall rules stopping direct egress from workload nodes, NetworkPolicy permitting
workloads to reach only the egress namespace, and authorization policies on the
gateway. That is a larger build than `ServiceEntry` alone, and it is what Option 2
means if it is to replace what Option 1 already enforces.

**The switch is mesh-wide.** On a managed control plane, `outboundTrafficPolicy` is
set in the `istio-<release-channel>` ConfigMap in `istio-system`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio-asm-managed-rapid
  namespace: istio-system
data:
  mesh: |-
    outboundTrafficPolicy:
      mode: REGISTRY_ONLY
```

One setting for every enrolled namespace. Switching it on before every application's
`ServiceEntry` exists breaks the ones that are missing, so adoption is all at once
rather than application by application. Whether a per-namespace equivalent is
supported is an open question with Google.

## What Cloud Foundry had, measured against both

Option 1 reproduces Cloud Foundry's two controls, per-application egress and
per-caller ingress, and tightens the second by requiring the caller to declare the
call as well. Nothing is lost.

Option 2 keeps both and adds encryption, workload identity, request-level control
and per-pair telemetry, none of which Cloud Foundry had. It trades the enforcement
point from the kernel to a proxy inside the workload, unless the full egress gateway
architecture is built alongside it.

## Implementation references

| Topic | Document |
|---|---|
| FQDN network policy, syntax and limits | https://cloud.google.com/kubernetes-engine/docs/how-to/fqdn-network-policies |
| Controlling egress with ServiceEntry, and the REGISTRY_ONLY switch | https://istio.io/latest/docs/tasks/traffic-management/egress/egress-control/ |
| `ServiceEntry` reference, including `exportTo` | https://istio.io/latest/docs/reference/config/networking/service-entry/ |
| `Sidecar` reference, including `outboundTrafficPolicy` and `egress.hosts` | https://istio.io/latest/docs/reference/config/networking/sidecar/ |
| Setting MeshConfig on a managed control plane | https://cloud.google.com/service-mesh/docs/enable-optional-features-managed |
| Egress gateway best practices, and why routing config is not a security boundary | https://cloud.google.com/service-mesh/docs/security/egress-gateways-best-practices |
| Egress gateway tutorial on GKE | https://cloud.google.com/service-mesh/docs/security/egress-gateway-gke-tutorial |
| Istio egress gateway task | https://istio.io/latest/docs/tasks/traffic-management/egress/egress-gateway/ |
| Feature request: FQDNNetworkPolicy with Cloud Service Mesh | https://issuetracker.google.com/issues/292142566 |
