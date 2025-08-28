# LFX architecture decisions

The architecture team has adopted Architecture Decision Records (ADRs) to
record and manage architectural decisions in a composable, maintainable way
for Linux Foundation engineering & devops.

## Usage

Teams should integrate all _Accepted_ decisions in their workflows via AI
tooling or other means.

When the engineering is adding new functionality or making significant changes,
technical design documents should reference any relevant ADRs that the work is
conforming to, by using the format `ADR-<number>`, e.g., `ADR-0001`.

Please note that _Accepted_ decisions are treated as stable. The architecture
team will introduce changes to existing decisions by marking them as
"Superseded" and creating a new decision.

## Key words

The capitalized key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", and "MAY"
distinguish between that which is required, recommended, or optional, as
defined in [RFC
2119](https://www.ietf.org/rfc/rfc2119.txt).

## Tooling and contributions

The architecture team uses [adr-tools](https://github.com/npryce/adr-tools) to
manage this repository.

Linux Foundation team members may inquire about proposed changes or
contributions in the #lfx-architecture Slack channel.

## License

Copyright The Linux Foundation and each contributor to LFX.

This project's source code is licensed under the MIT License. A copy of the
license is available in LICENSE.

This project's documentation is licensed under the Creative Commons Attribution
4.0 International License \(CC-BY-4.0\). A copy of the license is available in
LICENSE-docs.
