# 5. Machine credentials

Date: 2025-10-15

## Status

Accepted

## Context

Machine credentials differ from user credentials in that they ordinarily must be used by services to authenticate _non-interactively_—without the involvement of a person. This precludes machine credentials from leveraging multi-factor authentication (MFA), especially the phishing-resistant type, in the same way that theft of user credentials is mitigated.

Authorized staff members from the operations or development teams will, at some point in their duties, handle machine credentials from their workstations, including those under a "bring your own device" (BYOD) security practice. If we accept that all workstations have a _non-zero_ risk of being compromised (regardless of the competency or operational-security-practices of the workstation owner), combined with a _non-zero_ employee turnover rate (which further reduces our leverage or visibility for BYOD workstation security): we recognize that long-lived, MFA-free credentials represent exposure risk to a degree that must be mitigated by _reducing the scope of impact of leaked machine credentials_.

For the purpose of this decision, we define a _machine credential_ as any non-ephemeral credential (password, private key, API key, client secret, HMAC shared secret, etc.) which is used by a running component, service, or SaaS platform to authenticate itself in the absence of MFA, regardless of how it was issued by the validating party or service provider.

## Decision

Where alternatives exist to using machine credential, those MUST be used in favor of credential-based authentication. Some examples (not exhaustive):
- GitHub Actions OpenID Connect for authentication to 3rd-party platforms.
- Native service-role or execution-context features in cloud providers for authorization to cloud SDKs or cloud-hosted commodity services (databases or similar).
- Marketplace-style authorized connections between multiple SaaS platforms, where we provide consent, but where we are not provided with the credentials themselves.

Machine credentials SHOULD NOT be reused by discrete components, unless those components are tightly coupled and share a common runtime security boundary.

When possible, machine credentials SHOULD be restricted by IP to granular, LF-entitled IP ranges, such as:
- The runtime environment of the component(s) the credentials represent.
- VPN subnets of workstations of authorized LF staff, where the VPN uses phishing-resistant MFA for user authentication.

When possible, and there is no granular, LF-entitled IP range, machine credentials SHOULD by restricted to service-wide SaaS provider IP ranges.

Any machine credentials that are not IP restricted to granular, LF-entitled IP ranges MUST implement automated token rotation.

Credentials using automated token rotation MUST be automatically rotated no less frequently than every 2 months.

Credentials not using automated token rotation MUST be rotated no less frequently than every 2 years.

Components authenticating with OAuth2 client credentials (which includes both the `client_credentials` grant-type for machine-authorized access tokens, as well as privileged (non-public) client authentication during `authorization_code` grant flows), SHOULD use Private Key JWT client authentication, rather than a "client secrets", in order to support zero-impact credential rotations with staggered keys.

## Consequences

With the exception of automated rotation requirements, this decision is not new, but rather reinforces and republishes previous internal guidance on IP restrictions for machine credentials, while also setting more firm expectations on credential rotation schedules to reinforce existing ad hoc team practices.

As such, this decision requires teams to:

- Audit existing machine credential usage and implement IP restrictions where possible (and where not already implemented).
- Implement automated rotation systems for credentials that cannot be IP-restricted.
- Continue migrating from client secrets to [Private Key JWT authentication](https://github.com/linuxfoundation/lfx-architecture/blob/main/best-practices/private-key-client-auth.md) for OAuth2 implementations.
- Review and update team processes for regular manual rotation of credentials that cannot be automated.

Teams may need to invest in tooling and processes to support these requirements, but doing so will significantly protect us from costly response procedures to incidents like the [January 2023 CircleCI credentials exfiltration](https://circleci.com/blog/jan-4-2023-incident-report/) and improve our overall security posture.
