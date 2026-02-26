# Access Tokens RFC

## Problem

We are introducing a Deputy proxy binary that will send telemetry to the platform for potentially high volume workloads.

Telemetry has a provenance (who is sending it) and a destination (which workspace does it belong to). We need a mechanism to enforce that for a given workspace, only authorized proxies can produce telemetry.

Ideally, we would also like to know which instances of the proxy, which brings in questions of identity: what users are using which proxies?

Finally, it should also be possible to revoke access tokens.

## Solution A: long lived access tokens

We can implement long-lived access tokens, similar to what we did in Grafbase. The platform generates JWTs for itself, with a long expiration time that could span days to months.

Since we would also want to support revocation, we would need to keep state on which non-expired tokens are revoked.

There are two ways we can keep that state accessible to the Telemetry Sink:

1. **Database**: We can check postgres and hold an in-memory cache of recently checked tokens, like in the Grafbase Telemetry Sink.
2. **S3**: We can store the revoked tokens in an S3 bucket. This provides a scalable and durable storage solution for the revoked tokens, at the cost of a little extra bookkeeping. This is the object-storage approach in Grafbase.

In order to avoid having to check permissions, since these would be only for the proxy, these would be called **proxy access tokens** and given permissions to send telemetry to a single workspace, and nothing more for now. With more general access tokens, we would need to either store permissions in the token's claims (they would then not be editable), or get them from Postgres / S3, but we would then have to wait for the token to be fetched to know what it can do.

### Alternative

The recommended approach in OAuth is to use the client credentials flow, where the platform would issue client key / client secret pairs. The client credentials are in turn exchanged for a short lived access token. These tokens expire in general after 10 to 30 minutes. This makes revocation and updates of permissions not instant, but stateless.

### GraphQL API

```graphql
extend type Mutation {
    proxyAccessTokenCreate(input: ProxyAccessTokenCreateInput!): ProxyAccessTokenCreatePayload
    proxyAccessTokenDelete(id: ID!): ProxyAccessTokenDeletePayload
}

extend type Workspace {
    """
    Should be paginated in the actual implementation.
    """
    proxyAccessTokens: [ProxyAccessToken!]!
}

type ProxyAccessToken {
    createdAt: DateTime!
    createdBy: User
    id: ID!
    expiresAt: DateTime!
}

union ProxyAccessTokenCreatePayload = ProxyAccessTokenCreateSuccess | Forbidden
union ProxyAccessTokenDeletePayload = ProxyAccessTokenDeleteSuccess | Forbidden

input ProxyAccessTokenCreateInput {
    workspaceId: ID!
    """
    Optional.
    """
    expiresAt: DateTime
}

input ProxyAccessTokenCreateSuccess {
    accessToken: String!
    expiresAt: DateTime!
}

input ProxyAccessTokenCreateError {
    message: String!
}
```

## Solution B: machine identities

In this approach, we would not generate any secrets. We would allow configuring what identities (subjects) from which providers (issuers) have which permissions. From the platform's perspective, it is only configuration. At runtime, the proxy can exchange ID tokens for short lived access tokens — 15 minutes is a good default against the platform's token exchange endpoint. Here we care less about the token, and more about who presents it.

In enterprises that do have a machine identity framework (for example Azure AD, Okta, etc., but also mapped to k8s service accounts), users of the platform configure exactly how these identities, with different attributes, are granted which rights in the platform. In that scenario, **the platform is not issuing or checking any secret**: it checks the identity of the machine against its issuer's JWKS. The trust relationship is simply configured.

This is the same principle as OIDC workload identity in CI (GitHub, CircleCI).

With short-lived tokens, we don't need stateful revocation. Users configure what identities can or cannot send telemetry, and the next time an excluded identity's ID token is presented for exchange to a short lived access token, the exchange will fail.

### GraphQL API

```graphql
extend type Mutation {
    workspaceAddTrustRelationship(input: WorkspaceAddTrustRelationShipInput!): WorkspaceAddTrustRelationShipPayload
    # Delete and edit mutations follow the same idea.
}

input WorkspaceAddTrustRelationShipInput {
    workspaceID: ID!
    trustRelation: TrustRelation!
}

input TrustRelation {
    issuer: String!
    conditions: [TrustRelationClaimCondition!]
    entitlements: [Entitlement!]
}

enum Entitlement { PROXY_TELEMETR }

input TrustRelationClaimCondition @oneOf {
    matches: TrustRelationClaimConditionMatches!
    contains: TrustRelationClaimConditionContains!
    # etc. should also include negation
}

input TrustRelationClaimConditionMatches {
    claim: String!
    value: String!
}

input TrustRelationClaimConditionContains {
    claim: String!
    value: String!
}

```

## Conclusion

Short term, we should implement Solution A for the fastest getting started experience, similar to GitHub and other platforms. Longer term, enterprises will likely want something more like solution B. We can progressively get there, use case by use case — telemetry ingestion is just one of them.
