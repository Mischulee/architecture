---
Status: Proposed
Owner: HyperFleet Architecture Team
Last Updated: 2026-10-06
---

# 0029 - Cross-Cluster Applier Identity and Partition Binding

## Context

[ADR-0022](0022-api-mediated-desire-store-access.md) requires remote Appliers to use the Hub API rather than connect directly to Postgres. They are a new gateway caller and need access to exactly one partition without receiving Hub or database credentials.

## Decision

Remote Appliers use projected Kubernetes service-account JWTs issued by their management cluster. The Hub validates the registered issuer, audience, and service-account subject. They are a distinct, partition-scoped caller class; ADR-0020 remains unchanged.

The Hub operator manages AuthConfig entries mapping each remote identity `(issuer, subject, audience)` to exactly one partition. The verified AuthConfig entry, not a partition claim in the presented management-cluster token, determines the partition. The operator controls onboarding, replacement, and disablement.

Authorino validates the management-cluster token and issues an internal signed Hub Wristband with caller and partition claims. Envoy forwards the Wristband to the API, which validates it and authorizes from its claims, not client-supplied partition headers. The Wristband is used only between the gateway and API.

Binding revocation must have a defined maximum delay and an emergency invalidation path independent of cache expiry. The detailed design will define the cache TTL upper bound and invalidation mechanism.

The detailed design will specify gateway and API-side token-validation configuration and authorization rules.

## Consequences

**Gains:** Remote Appliers receive refreshed local credentials; partition ownership and revocation are centrally controlled; the existing Envoy/Authorino security model remains in use through the dedicated Desire Store gateway.

**Trade-offs:** The design depends on operator reconciliation and AuthConfig propagation, revocation is bounded by gateway cache lifetime, and the new caller class requires additional AuthConfig.

## Alternatives Considered

| Alternative | Why Rejected |
|-------------|--------------|
| Separate identity-to-partition registry | Adds a runtime lookup, cache, and binding-management dependency that operator-managed AuthConfig avoids. |
| OCI workload identity | Couples the caller model to one cloud IAM system and does not provide a portable management-cluster identity model. |
| Hub-minted bearer credentials | Requires a separate Hub credential-issuance and renewal service and distributes Hub-issued credentials to management clusters. |
| Partition in a management-cluster token claim | Couples partition authorization to a claim controlled by the management-cluster issuer instead of the Hub's AuthConfig mapping. |
| Deterministic issuer or subject mapping | Couples partition identity to issuer or subject naming and makes identity changes harder to manage. |

## References

- [Desire Store HLD](../docs/desire-store-hld.md)
- [ADR-0020: Envoy and Authorino as the API Authentication Gateway](0020-envoy-authorino-api-gateway.md)
- [ADR-0022: API-Mediated Desire Store Access](0022-api-mediated-desire-store-access.md)
- [Remote Applier Connectivity and Partition-Scoped Access to Postgres spike](../docs/spike-remote-applier-postgres-access.md)
- [HYPERFLEET-1737](https://redhat.atlassian.net/browse/HYPERFLEET-1737)
- [HYPERFLEET-1645](https://redhat.atlassian.net/browse/HYPERFLEET-1645)
