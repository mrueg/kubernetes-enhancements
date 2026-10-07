# KEP-NNNN: Multiple Application Protocols and ALPN for Service Ports

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1: gRPC and HTTP on one port](#story-1-grpc-and-http-on-one-port)
    - [Story 2: Interoperability between implementations](#story-2-interoperability-between-implementations)
    - [Story 3: Multi-protocol traffic between pods](#story-3-multi-protocol-traffic-between-pods)
    - [Story 4: Publishing HTTPS/SVCB DNS records](#story-4-publishing-httpssvcb-dns-records)
    - [Story 5: Backend TLS with the right ALPN](#story-5-backend-tls-with-the-right-alpn)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [API](#api)
  - [Semantics](#semantics)
  - [ALPN scope](#alpn-scope)
  - [Validation](#validation)
  - [Warnings](#warnings)
  - [Keeping <code>appProtocol</code> and <code>appProtocols</code> in sync](#keeping-appprotocol-and-appprotocols-in-sync)
  - [EndpointSlice](#endpointslice)
  - [Feature gate handling](#feature-gate-handling)
  - [Consumers](#consumers)
  - [Examples](#examples)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [A new <code>applicationHints</code> field](#a-new-applicationhints-field)
  - [A flat <code>appProtocols []string</code>](#a-flat-appprotocols-string)
  - [A per-port <code>alpn</code> list](#a-per-port-alpn-list)
  - [Allow the same port to appear multiple times](#allow-the-same-port-to-appear-multiple-times)
  - [Comma-separated <code>appProtocol</code>](#comma-separated-appprotocol)
  - [Annotations or Gateway API only](#annotations-or-gateway-api-only)
- [Future Work](#future-work)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) within one minor version of promotion to GA
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

`ServicePort.appProtocol` ([KEP-1507](/keps/sig-network/1507-app-protocol))
lets a Service owner tell implementations (ingress and Gateway controllers,
service meshes, cloud load balancers, DNS providers) which application
protocol a port speaks. It holds exactly one value. This KEP is mainly about
interoperability, which a single value cannot provide when:

- one port serves several protocols, for example gRPC and HTTP/1.1, or
  WebSocket and HTTP/2;
- different implementations in the same cluster understand different names
  for the same protocol (the interoperability problem that
  [KEP-3726](/keps/sig-network/3726-standard-application-protocols) named as
  follow-up work).

These problems affect traffic at the edge (Gateways, cloud load balancers)
and inside the cluster (service meshes, Gateway API for mesh traffic, L7
policy and observability tools).

As a secondary benefit, each protocol can also declare the TLS [ALPN]
protocol IDs the backend accepts. Gateways and load balancers can use this to
pick the protocol for TLS connections to the backend, and external DNS
providers can use it to publish [HTTPS/SVCB DNS records][RFC9460] with an
`alpn` parameter. Whether ALPN belongs in this KEP is an
[open question](#alpn-scope).

This KEP adds an optional, ordered list `appProtocols` to `ServicePort` and to
the EndpointSlice `EndpointPort`. Each entry has a `name` (same syntax and
meaning as `appProtocol`) and an optional list of `alpn` protocol IDs. The
existing `appProtocol` field stays and is kept in sync with the first list
entry, using the same two-field pattern Kubernetes already uses for
`clusterIP` and `clusterIPs`.

Tracking issue: [kubernetes/kubernetes#118407](https://github.com/kubernetes/kubernetes/issues/118407).

[ALPN]: https://www.rfc-editor.org/rfc/rfc7301
[RFC9460]: https://www.rfc-editor.org/rfc/rfc9460

## Motivation

`appProtocol` was introduced as a hint so implementations can "offer richer
behavior for protocols that they understand". Since then:

1. **Multi-protocol ports are common.** gRPC servers often expose
   grpc-gateway (HTTP/1.1 + JSON) on the same port; etcd does this. Knative and
   Cloud Run restrict workloads to a single port and need to tell ingress that
   the port serves both gRPC and HTTP. Envoy-based proxies upgrade WebSocket
   over HTTP/2 on the same port. A single value forces the owner to pick one and
   lose behavior for the other.
2. **Implementations disagree on names.** KEP-3726 introduced
   `kubernetes.io/h2c`, `kubernetes.io/ws` and `kubernetes.io/wss`, but many
   implementations still use their own prefixed names or bare values like
   `HTTP2`. With a single value, a Service cannot satisfy two implementations
   that both run in the cluster (KEP-3726 cites GKE and Istio).

Both problems apply to traffic between pods as well as to traffic at the
edge. A mesh that classifies a gRPC + HTTP/1.1 port as only one protocol
either treats the other as opaque TCP (no L7 routing, retries or per-request
metrics) or falls back to protocol sniffing, which adds latency and can guess
wrong. L7 network policy and observability tools have the same problem when
choosing a protocol parser.

A secondary problem is that **ALPN cannot be expressed.**
`appProtocol: https` does not say whether the backend accepts `h2`,
`http/1.1`, or both, or whether an `h3` (QUIC) endpoint exists on the
matching UDP port. Components that must decide before a connection exists
need this information, for example to:

- choose the protocol and ALPN for a new TLS connection to the backend
  (Gateway API `BackendTLSPolicy`, internal gateways, cloud load balancers).
  gRPC over TLS requires `h2` to be negotiated;
- route non-HTTP TLS protocols that are multiplexed on one port by ALPN,
  such as `acme-tls/1` next to `h2`;
- publish HTTPS/SVCB records (RFC 9460) for externally exposed Services,
  which carry an `alpn` SvcParam. external-dns has an open request for this
  ([kubernetes-sigs/external-dns#4878](https://github.com/kubernetes-sigs/external-dns/issues/4878))
  and Gateway API has discussed exposing ALPN for the same reason
  ([kubernetes-sigs/gateway-api#3849](https://github.com/kubernetes-sigs/gateway-api/issues/3849)).

TLS negotiates ALPN on every connection, so declaring it in the API does not
help two endpoints that talk to each other directly.

### Goals

- Let a Service port declare more than one application protocol, in order of
  preference.
- Let each application protocol optionally declare the TLS ALPN protocol IDs
  it is served with.
- Keep `appProtocol` working unchanged for existing clients and consumers.
- Carry the same information on EndpointSlices, so consumers that only watch
  EndpointSlices see it.
- Make the ALPN information available to external DNS providers (for
  example external-dns) so they can build HTTPS/SVCB records.

### Non-Goals

- Changing kube-proxy or any data-path behavior. Like `appProtocol`, the new
  field is a hint for higher-level implementations.
- Adding HTTPS/SVCB records to the
  [Kubernetes DNS specification](https://github.com/kubernetes/dns/blob/master/docs/specification.md).
  This KEP makes that possible; specifying it is [future work](#future-work).
- Adding `appProtocol(s)` to `ContainerPort`.
- Adding the new field to the deprecated core `Endpoints` API
  ([KEP-4974](/keps/sig-network/4974-deprecate-endpoints)).
- Defining new `kubernetes.io/` protocol names. KEP-3726 owns that list.
- Validating that a backend actually speaks the declared protocols.

## Proposal

Add `appProtocols` to `core/v1.ServicePort` and `discovery/v1.EndpointPort`:

```yaml
ports:
- name: https
  port: 443
  protocol: TCP
  appProtocol: https            # kept, mirrors appProtocols[0].name
  appProtocols:
  - name: https
    alpn: ["h2", "http/1.1"]
  - name: example.com/grpc-web
- name: https-quic
  port: 443
  protocol: UDP
  appProtocols:
  - name: https
    alpn: ["h3"]
```

### User Stories

#### Story 1: gRPC and HTTP on one port

As a developer running a gRPC service with grpc-gateway on port 8080, I want
my Gateway controller to route both gRPC (HTTP/2 cleartext) and plain
HTTP/1.1 traffic to that port, so I don't need two ports or two Services.

```yaml
appProtocols:
- name: kubernetes.io/h2c
- name: http
```

#### Story 2: Interoperability between implementations

As a cluster operator running a mesh and a cloud load balancer that use
different names for HTTP/2, I want to list both names so that both understand
the port, without annotations.

```yaml
appProtocols:
- name: kubernetes.io/h2c
- name: mesh.example.com/http2
```

#### Story 3: Multi-protocol traffic between pods

As a developer whose pods call a Service in the same cluster through a service
mesh, I want the mesh to apply HTTP routing, retries and per-request metrics
to both the gRPC and the HTTP/1.1 traffic on that port. The same applies to
Gateway API routes attached to the Service for mesh traffic (GAMMA), and to
L7 network policy or observability tools that pick a protocol parser for the
port. Today they see one protocol and treat the other as opaque TCP or guess.

#### Story 4: Publishing HTTPS/SVCB DNS records

As a platform engineer using external-dns or a cluster DNS plugin, I want to
publish an HTTPS record for my `LoadBalancer` Service that advertises
`alpn="h3,h2"`, so browsers can connect over HTTP/3 without an Alt-Svc
round trip. The DNS provider reads `appProtocols[*].alpn` from the TCP and
UDP ports that share port number 443.

#### Story 5: Backend TLS with the right ALPN

As a Gateway implementer, I want to know whether a TLS backend accepts `h2`
so I can offer it in the ALPN extension when re-encrypting, instead of
falling back to HTTP/1.1 or guessing.

### Notes/Constraints/Caveats

- **ALPN is per transport.** ALPN is negotiated on a TLS (or QUIC)
  connection to a specific port and transport protocol. `h3` runs over QUIC,
  so it belongs on a `protocol: UDP` port. HTTPS/SVCB records describe an
  endpoint, not a transport, so a DNS provider must combine the TCP and UDP
  ports that share a port number. This KEP does not merge them in the API.
- **ALPN is optional and meaningless for cleartext protocols.** It only
  applies when the named application protocol runs over TLS or QUIC.
- **The new field is still a hint.** Consumers that do not understand an
  entry ignore it, just as they do today with `appProtocol`.
- **HTTPS/SVCB records bring little inside the cluster.** The ALPN data could
  later be published by cluster DNS, but it would help pod-to-pod traffic
  very little. In-cluster round trips are short, so skipping a handshake
  saves almost nothing. Most in-cluster clients (Go, gRPC, Envoy) do not
  query HTTPS/SVCB records, and `alpn="h2,http/1.1"` adds nothing that TLS
  does not already negotiate. The DNS benefit is for external clients,
  mainly browsers, reaching externally exposed Services. The in-cluster
  benefit of this KEP comes from meshes, gateways and policy tools reading
  `appProtocols`, not from DNS.

### Risks and Mitigations

| Risk | Mitigation |
| --- | --- |
| Two fields describing the same thing drift apart and confuse consumers. | API server enforces `appProtocol == appProtocols[0].name` and fills either from the other ([sync rules](#keeping-appprotocol-and-appprotocols-in-sync)). This is the `clusterIP`/`clusterIPs` pattern. |
| Old clients doing read-modify-write drop the unknown list. | Update logic restores the list from the stored object when the client leaves `appProtocol` unchanged. |
| EndpointSlice churn on upgrade. | The list is defaulted on read, so the stored and desired slices compare equal for Services that only use `appProtocol`. |
| Object size growth. | At most 8 protocols per port and 8 ALPN IDs per protocol. Typical use is 1 to 3 short strings. |
| Users put ALPN IDs on the wrong port (`h3` on TCP). | API warnings, not errors (see [Warnings](#warnings)). |

## Design Details

### API

`core/v1`:

```go
type ServicePort struct {
	// ... existing fields ...

	// The application protocol for this port.
	// ...existing documentation...
	// When appProtocols is set, this field must be equal to
	// appProtocols[0].name. If it is unset and appProtocols is set, it is
	// set to appProtocols[0].name.
	// +optional
	AppProtocol *string `json:"appProtocol,omitempty"`

	// AppProtocols lists the application protocols served on this port, in
	// order of preference. It is a hint for implementations, which should
	// use the first entry they understand and ignore the others.
	// Each name follows the same rules as appProtocol.
	// If appProtocol is set and this field is not, it defaults to a single
	// entry whose name is appProtocol.
	// At most 8 entries are allowed, and names must be unique.
	// This field is alpha-level and is only honored by servers that enable
	// the ServicePortAppProtocols feature gate.
	// +featureGate=ServicePortAppProtocols
	// +listType=atomic
	// +optional
	AppProtocols []AppProtocol `json:"appProtocols,omitempty"`
}

// AppProtocol describes one application protocol served on a port.
type AppProtocol struct {
	// Name of the application protocol. Valid values are the same as for
	// ServicePort.appProtocol: an IANA standard service name, a
	// Kubernetes-defined name such as 'kubernetes.io/h2c', or an
	// implementation-defined prefixed name such as
	// 'mycompany.com/my-custom-protocol'.
	// +required
	Name string `json:"name"`

	// ALPN lists the TLS Application-Layer Protocol Negotiation protocol IDs
	// (RFC 7301) the backend accepts for this application protocol, in order
	// of preference. Values should come from the IANA "TLS
	// Application-Layer Protocol Negotiation (ALPN) Protocol IDs" registry,
	// for example "h2", "http/1.1" or "h3". Only meaningful when the
	// application protocol runs over TLS or QUIC on this port's transport
	// protocol. At most 8 entries are allowed.
	// +listType=atomic
	// +optional
	ALPN []string `json:"alpn,omitempty"`
}
```

`discovery/v1`:

```go
type EndpointPort struct {
	// ... existing fields ...

	// appProtocols mirrors ServicePort.appProtocols, with the same
	// validation and the same relationship to appProtocol.
	// +featureGate=ServicePortAppProtocols
	// +listType=atomic
	// +optional
	AppProtocols []corev1.AppProtocol `json:"appProtocols,omitempty"`
}
```

`discovery/v1` already imports `core/v1` types (`ObjectReference`,
`Protocol`), so the struct is shared instead of duplicated.

The list is `atomic`, not a map keyed by `name`, because order is part of the
meaning and the first entry has to match `appProtocol`.

### Semantics

- The list is the set of application protocols the port serves. Order is the
  Service owner's preference.
- A consumer should use the first entry it understands. Consumers that need
  one value and do not understand the list keep reading `appProtocol`, which
  is always the first entry.
- `alpn` on an entry lists the ALPN IDs accepted when that protocol is used
  over TLS or QUIC on this port. The ALPN set for the port as a whole is the
  ordered union over all entries.
- Leaving `alpn` empty means "not specified", not "no ALPN". Consumers keep
  their current defaults.

### ALPN scope

<<[UNRESOLVED sig-network: should alpn be part of alpha?]>>

The protocol list (`name`) solves the main interoperability problems on its
own. `alpn` is the part most likely to be questioned: it exposes TLS detail in
the core Service API, and some of its use cases could live in Gateway API
(`BackendTLSPolicy`) instead.

Options:

1. Ship `name` and `alpn` together in alpha (current proposal).
2. Ship only `name` in alpha and add `alpn` in a later stage or a separate
   KEP. Because each entry is a struct, `alpn` can be added later without an
   API break.
3. Leave ALPN to Gateway API.

<<[/UNRESOLVED]>>

### Validation

When `appProtocols` is set:

- 1 to 8 entries.
- `name` is required, must pass the existing `appProtocol` validation, and
  must be unique within the port.
- `alpn` has at most 8 entries. Each entry is 1 to 255 bytes (the RFC 7301
  limit), unique within the entry, and made of printable ASCII
  (`0x21`–`0x7E`) excluding `,` and `\`. The exclusions keep values
  representable in the SVCB presentation format without escaping. Every
  protocol ID in the IANA registry satisfies this.
- If `appProtocol` is also set, it must equal `appProtocols[0].name`.

Validation is the same for `ServicePort` and `EndpointPort`. Updates use
ratcheting: if a port's `appProtocols` is unchanged from the stored object, it
is not revalidated against tighter rules added later.

### Warnings

The API server returns
[warnings](https://kubernetes.io/blog/2020/09/03/warnings/) (not errors) for
likely mistakes:

- `h3` or another QUIC-only ID (`h3-*`, `doq`) on a port with `protocol: TCP`
  or `SCTP`;
- a TLS-over-TCP ID (`h2`, `http/1.1`, etc.) on a port with `protocol: UDP`;
- `alpn` set on a known cleartext protocol (`http`, `kubernetes.io/h2c`,
  `kubernetes.io/ws`).

### Keeping `appProtocol` and `appProtocols` in sync

This follows the approach used for `clusterIP`/`clusterIPs` and
`ipFamilies`.

**Defaulting (create, update, and read from storage).** In
`SetDefaults_Service` and the `EndpointSlice` defaulter, for every port:

| `appProtocol` | `appProtocols` | Result |
| --- | --- | --- |
| unset | unset | unchanged |
| set | unset | `appProtocols = [{name: appProtocol}]` |
| unset | set | `appProtocol = appProtocols[0].name` |
| set | set | unchanged; validation requires them to agree |

Because defaulting also runs on decode, existing objects stored before the
upgrade show a one-entry `appProtocols` to clients, and every reader sees the
same form.

**Update (old clients).** Clients built before this field drop it during
read-modify-write. In `PrepareForUpdate`, ports are matched between the old
and new object by `(port, protocol)`, which is already the port's list key.
Before defaulting, for each matched port:

1. New `appProtocols` empty and `appProtocol` equal to the old value: the
   client did not touch the protocol, so copy `appProtocols` from the old
   object.
2. New `appProtocols` equal to the old value and `appProtocol` changed (an
   old client patching only the scalar): replace the first entry with
   `{name: <new appProtocol>}` (without ALPN, since that described the old
   protocol), remove any later entry with the same name, and keep the rest.
3. New `appProtocols` equal to the old value and `appProtocol` cleared: clear
   `appProtocols`.
4. Anything else: the client set the list on purpose, so apply the normal
   defaulting and validation.

### EndpointSlice

- The EndpointSlice controller copies `appProtocols` from the `ServicePort`
  to the `EndpointPort`, like it copies `appProtocol` today.
- Port grouping in the controller hashes `EndpointPort`. The new field is
  added to that hash. Both the desired and the existing slice are defaulted
  the same way, so Services that only use `appProtocol` hash the same as
  before, and the upgrade does not rewrite slices.
- The EndpointSlice mirroring controller only sees `Endpoints.appProtocol`.
  Mirrored slices get the one-entry list through defaulting.
- The core `Endpoints` API is not changed.

### Feature gate handling

Gate `ServicePortAppProtocols`, on kube-apiserver and
kube-controller-manager.

- Gate disabled in kube-apiserver: `appProtocols` is dropped on create, and
  on update unless the stored object already used it (the standard
  `dropDisabledFields` pattern). Read-defaulting of the list is off, so
  nothing changes for clients.
- Gate disabled in kube-controller-manager: the EndpointSlice controller does
  not copy the list. The API server still defaults a one-entry list from
  `appProtocol` on the slice if its own gate is on.

### Consumers

None of these changes are part of this KEP. They are listed to show that the
design fits the consumers it is meant for.

- **Gateway API**: GEP-1911 (backend protocol selection) would read the first
  entry the implementation understands. `BackendTLSPolicy` could take ALPN
  from the list. This covers both Gateways and routes attached to Services
  for mesh traffic (GAMMA).
- **Service meshes**: classify multi-protocol ports correctly for traffic
  between pods instead of picking one protocol or sniffing.
- **L7 network policy and observability tools**: pick the right protocol
  parser for a port from a standard source instead of vendor-specific
  resources.
- **Multi-Cluster Services API**: `ServiceImport` ports copy `appProtocol`
  from exported Services and would copy `appProtocols` the same way.
- **external-dns / cluster DNS**: build an HTTPS record for port 443 from
  the ordered union of `alpn` on the TCP and UDP ports with that number.
  Build an SVCB record with `port=` for other ports.
- **Cloud providers**: choose the backend protocol (HTTP/2, gRPC) from the
  list instead of from annotations.
- **kubectl describe**: show the list for each port.

### Examples

A Service on 443 with HTTP/1.1, HTTP/2 and HTTP/3:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - name: https
    port: 443
    protocol: TCP
    appProtocols:
    - name: https
      alpn: ["h2", "http/1.1"]
  - name: https-quic
    port: 443
    protocol: UDP
    appProtocols:
    - name: https
      alpn: ["h3"]
```

After defaulting, both ports also show `appProtocol: https`. A DNS provider
could publish:

```
web.example.com. 300 IN HTTPS 1 . alpn="h3,h2,http/1.1"
```

An old client that only knows `appProtocol`:

```yaml
ports:
- port: 8080
  appProtocol: kubernetes.io/h2c
```

reads back as:

```yaml
ports:
- port: 8080
  appProtocol: kubernetes.io/h2c
  appProtocols:
  - name: kubernetes.io/h2c
```

### Test Plan

[x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

None.

##### Unit tests

- `pkg/apis/core/validation`: field validation, uniqueness, ALPN byte
  rules, agreement with `appProtocol`, ratcheting.
- `pkg/apis/core/v1`: defaulting table above, including on read.
- `pkg/apis/discovery/validation` and `pkg/apis/discovery/v1`: same for
  `EndpointPort`.
- `pkg/registry/core/service`: `PrepareForCreate`/`PrepareForUpdate` sync
  rules for old clients, `dropDisabledFields`, and warnings.
- `pkg/registry/discovery/endpointslice`: `dropDisabledFields`.
- `pkg/controller/endpointslice` and
  `staging/src/k8s.io/endpointslice`: list is copied, port hashing is stable
  for Services without the list.

##### Integration tests

- `test/integration/service`: create and update with the gate on and off,
  including an old-client round trip (decode into a struct without the field,
  update `appProtocol`, verify the result).
- Gate enabled, then disabled, then enabled again: stored values are kept and
  can still be updated.
- `test/integration/endpointslice`: the controller propagates the list and
  does not rewrite slices on upgrade.

##### e2e tests

- `test/e2e/network`: create a Service with `appProtocols`, verify
  `appProtocol` is defaulted and that EndpointSlices carry the same list.
  Update the list and verify propagation.

### Graduation Criteria

#### Alpha

- Feature gate implemented, off by default.
- API, defaulting, validation, warnings and sync logic implemented with unit
  and integration tests.
- EndpointSlice controller support.

#### Beta

- Gate on by default.
- At least one implementation (Gateway API controller, mesh, cloud provider,
  or DNS provider) consumes the field and gives feedback.
- e2e tests in place and stable.
- kubectl describe support.
- Documentation on kubernetes.io. Decision recorded on whether to add HTTPS/SVCB
  records to the Kubernetes DNS specification.

#### GA

- At least two implementations consume the field.
- Conformance tests for the API behavior.
- No unresolved issues from beta.

### Upgrade / Downgrade Strategy

- Upgrade: no action required. Existing Services read back with a one-entry
  list.
- Downgrade: older API servers ignore the field. It stays in etcd and becomes
  visible again after re-upgrade, unless an old client rewrote the object
  without it. `appProtocol` keeps working throughout.

### Version Skew Strategy

- **New API server, old kube-controller-manager:** EndpointSlices get only
  `appProtocol`. The API server defaults a one-entry list, so consumers lose
  the extra entries until kube-controller-manager is upgraded.
- **Old API server, new kube-controller-manager:** the controller sets the
  list but the old API server drops it. No harm.
- **Old clients:** handled by the [update rules](#keeping-appprotocol-and-appprotocols-in-sync).
- **Consumers:** consumers that do not know the field keep using
  `appProtocol`.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: `ServicePortAppProtocols`
  - Components depending on the feature gate: kube-apiserver,
    kube-controller-manager

###### Does enabling the feature change any default behavior?

Clients reading Services and EndpointSlices that have `appProtocol` set also
see a one-entry `appProtocols` list. Nothing else changes.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. Disabling the gate stops read-defaulting and rejects new uses. Objects
that already have the field keep it and can still be updated.

###### What happens if we reenable the feature if it was previously rolled back?

Stored values become visible again. Objects updated by clients that did not
send the field while the gate was off have only `appProtocol`, which defaults
back to a one-entry list.

###### Are there any tests for feature enablement/disablement?

Yes, unit tests for `dropDisabledFields` and an integration test that toggles
the gate (see [Test Plan](#test-plan)).

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?

The field is a hint and is not used in the data path, so running workloads
are not affected. The main risk is a sync bug that changes `appProtocol` on
existing Services. Consumers that act on `appProtocol` (load balancers,
meshes) could then change backend protocol. The update rules and their
integration tests address this.

###### What specific metrics should inform a rollback?

An increase in Service or EndpointSlice update rates
(`apiserver_request_total{resource=~"services|endpointslices",verb="update"}`)
or in `endpoint_slice_controller_changes` right after enabling the gate.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

To be done before beta.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

No. `appProtocol` is not deprecated.

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?

Look for Services with more than one entry, or with ALPN set:

```
kubectl get svc -A -o json | jq '.items[] | select(any(.spec.ports[]; (.appProtocols|length) > 1 or any(.appProtocols[]?; .alpn)))'
```

###### How can someone using this feature know that it is working for their instance?

- [x] API .spec
  - Other field: `spec.ports[*].appProtocols` on the Service and
    `ports[*].appProtocols` on its EndpointSlices.

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

Existing API server and EndpointSlice controller SLOs. No new ones.

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

- [x] Metrics
  - Metric name: `apiserver_request_duration_seconds`,
    `endpoint_slice_controller_endpointslices_changed_per_sync`
  - Components exposing the metric: kube-apiserver, kube-controller-manager

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

No.

### Dependencies

###### Does this feature depend on any specific services running in the cluster?

No. It is only useful once a consumer reads the field.

### Scalability

###### Will enabling / using this feature result in any new API calls?

No. Updating the list causes the same Service and EndpointSlice updates as
updating `appProtocol` does today.

###### Will enabling / using this feature result in introducing new API types?

No new kinds. A new struct `AppProtocol` is added.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

Yes. `Service` and `EndpointSlice` grow per port. Services that only use
`appProtocol` add about 20 bytes per port when serialized. The worst case is
8 entries with 8 × 255-byte ALPN IDs, about 16 KiB per port. Realistic use is
under 100 bytes per port.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

No measurable increase.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

No.

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

No. Nothing runs on nodes.

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?

Same as any other field. Nothing new.

###### What are other known failure modes?

- `appProtocol` and `appProtocols[0].name` disagree:
  - Detection: create or update is rejected with a validation error naming
    both fields.
  - Mitigations: set only one of the fields, or make them agree.
  - Diagnostics: the API error message.
  - Testing: unit tests.
- A consumer ignores the list:
  - Detection: the consumer's behavior only reflects `appProtocol`.
  - Mitigations: put the protocol that consumer needs first.
  - Diagnostics: consumer-specific.
  - Testing: N/A.

###### What steps should be taken if SLOs are not being met to determine the problem?

Check API server and EndpointSlice controller metrics for unusual update
volume on Services and EndpointSlices, then disable the gate.

## Implementation History

- 2023-06-02: [kubernetes/kubernetes#118407](https://github.com/kubernetes/kubernetes/issues/118407) opened.
- 2025-04-15: ALPN-aware structure proposed in that issue.
- 2026-10-06: KEP created as `provisional`.

## Drawbacks

- Two fields that describe the same thing, with sync rules that consumers do
  not see but API reviewers must maintain. The `clusterIP`/`clusterIPs`
  precedent shows this works but is not free.
- ALPN is TLS-specific. Putting it in a generic Service API exposes
  transport detail that some reviewers may prefer to keep in Gateway API.

## Alternatives

### A new `applicationHints` field

Proposed in
[kubernetes/kubernetes#118407](https://github.com/kubernetes/kubernetes/issues/118407#issuecomment-2807488328):
a list of `{protocol, alpn}` entries under a new name, with `appProtocol`
deprecated.

```yaml
applicationHints:
- protocol: https
  alpn: h3
- protocol: kubernetes.io/ws
```

Not chosen because:

- `appProtocol` is a GA `core/v1` field and cannot be removed, so the new
  field must stay in sync with it either way. Kubernetes names the list form
  of a single field with the plural (`clusterIP`/`clusterIPs`,
  `ipFamily`/`ipFamilies`), which makes the relationship clear. With a
  different name, nothing shows that `applicationHints[0].protocol` must
  equal `appProtocol`.
- `appProtocol` is already a hint, so "hint" adds no meaning. A generic name
  also invites unrelated settings (timeouts, retries) to be added later.
- `protocol` inside an entry would clash with `ServicePort.protocol`
  (TCP/UDP/SCTP). This KEP uses `name`.
- Existing documentation, GEP-1911, KEP-3726 and implementations all use
  "appProtocol", so `appProtocols` is easier to find.

The main argument for a new name is that the entries carry more than a
protocol name (`alpn`). The entry type, `AppProtocol` with `name` and `alpn`
fields, covers that, and new attributes can be added to it later.

### A flat `appProtocols []string`

Simpler, and enough for stories 1 and 2. It cannot express ALPN, so stories 3
and 4 would need another field later, and a struct cannot be added to a list
of strings without a new field. Using a struct now leaves room for future
per-protocol attributes.

### A per-port `alpn` list

`ServicePort.alpn []string` next to `appProtocols []string`. This loses the
link between a protocol and its ALPN IDs, which matters when one port
multiplexes unrelated protocols by ALPN (for example `h2` and `acme-tls/1`).

### Allow the same port to appear multiple times

Relax the `(port, protocol)` uniqueness so a port can be listed once for each
`appProtocol`, similar to mixed-protocol Services
([KEP-1435](/keps/sig-network/1435-mixed-protocol-lb)). This breaks every
consumer that keys ports by `(port, protocol)` and makes kube-proxy and cloud
providers create duplicate rules.

### Comma-separated `appProtocol`

Breaks existing validation and every consumer that compares the value
directly.

### Annotations or Gateway API only

Annotations are not portable, which is why `appProtocol` exists. Gateway API
only covers traffic through a Gateway. It does not cover mesh traffic, cloud
load balancers for `type: LoadBalancer`, or DNS records for Services.

## Future Work

- Specify HTTPS/SVCB records in the Kubernetes DNS specification, for example
  an SVCB record at `_<port>._<proto>.<svc>.<ns>.svc.<zone>` carrying `alpn`
  and `port`, and implement it in CoreDNS.
- Use the list in Gateway API (GEP-1911, `BackendTLSPolicy`).
- Revisit `ContainerPort` if there is demand.
