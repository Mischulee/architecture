---
Status: Proposed
Owner: HyperFleet Architecture Team
Last Updated: 2026-10-05
---

# 0029 - Cross-Cluster Applier Identity and Partition Binding

## Context

[ADR-0022](0022-api-mediated-desire-store-access.md) requires remote Appliers to use the Hub API rather than connect directly to Postgres. They are a new gateway caller and need access to exactly one partition without receiving Hub or database credentials.

## Decision

Remote Appliers use short-lived projected Kubernetes service-account JWTs issued by their management cluster. The Hub validates a registered issuer, audience, and service-account subject. They are a distinct, partition-scoped caller class; ADR-0020 remains unchanged.

The Desire Store API independently validates the presented bearer JWT as defense-in-depth, consistent with ADR-0020. The gateway forwards the token and injects trusted caller and partition headers; the API treats the injected identity and partition as authoritative for authorization and data access.

The Hub-owned identity-to-partition registry maps each remote identity `(issuer, subject, audience)` to exactly one partition. A Hub-owned binding-management component controls binding creation, replacement, and disablement.

The detailed design will specify gateway and API-side token-validation configuration and authorization rules.

## Consequences

**Gains:** Remote Appliers receive refreshed local credentials; partition ownership and revocation are centrally controlled; the existing gateway boundary remains in use.

**Trade-offs:** The Hub depends on issuer and registry availability, revocation is bounded by gateway cache lifetime, and the new caller class requires additional AuthConfig.

## Alternatives Considered

| Alternative | Why Rejected |
|-------------|--------------|
| OCI workload identity | Couples the caller model to one cloud IAM system and does not provide a portable management-cluster identity model. |
| Hub-minted bearer credentials | Requires a separate Hub credential-issuance and renewal service and distributes Hub-issued credentials to management clusters. |
| Partition in a validated token claim | Couples partition authorization to issuer-controlled claims instead of a Hub-owned binding. |
| Static AuthConfig mapping | Adds per-cluster gateway configuration lifecycle for onboarding, replacement, and revocation. |
| Deterministic issuer or subject mapping | Couples partition identity to issuer or subject naming and makes identity changes harder to manage. |

## References

- [Desire Store HLD](../docs/desire-store-hld.md)
- [ADR-0020: Envoy and Authorino as the API Authentication Gateway](0020-envoy-authorino-api-gateway.md)
- [ADR-0022: API-Mediated Desire Store Access](0022-api-mediated-desire-store-access.md)
- [Remote Applier Connectivity and Partition-Scoped Access to Postgres spike](../docs/spike-remote-applier-postgres-access.md)
- [HYPERFLEET-1737](https://redhat.atlassian.net/browse/HYPERFLEET-1737)
- [HYPERFLEET-1645](https://redhat.atlassian.net/browse/HYPERFLEET-1645)
