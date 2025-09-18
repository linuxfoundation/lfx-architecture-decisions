# 4. Generative AI model and provider requirements

Date: 2025-09-10

## Status

Accepted

## Context

As Linux Foundation applications and services increasingly integrate Generative AI capabilities for inference and agentic use cases, there is a need to establish standardized requirements for model selection, provider authorization, and service architecture. This ensures security, compliance, monitoring, and cost control while maintaining consistency across the organization's AI-enabled applications.

## Decision

Linux Foundation applications and services processing user data via Generative AI MUST use an authorized model and provider, as defined by the Cloud Operations team, for inference or agentic use cases.

The application or service SHOULD use a LiteLLM proxy hosted and managed by the Cloud Operations team which curates access to authorized models and providers, and provides monitoring and analytics.

The application or service SHOULD use an abstraction library or framework (such as Agno, LangChain, or Semantic Kernel), provided it is distributed under a compatible license.

The application or service MUST NOT use a 3rd-party proxy service for AI model access.

## Consequences

Development teams will need to coordinate with the Cloud Operations team to ensure their chosen AI models and providers are authorized before implementation. This may introduce some delays in the development process but will provide better security oversight and cost control.

Using the centrally managed LiteLLM proxy will provide standardized monitoring and analytics across all AI-enabled applications, but teams will need to adapt their implementations to work with this centralized service.

The requirement to use abstraction libraries will improve portability and reduce vendor lock-in, but may require additional development effort to implement proper abstractions.

Prohibiting 3rd-party proxy services reduces security risks and ensures all AI traffic goes through approved channels, but may limit some teams' preferred tooling choices.
