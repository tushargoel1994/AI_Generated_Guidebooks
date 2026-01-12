# Python & AI Tools Reference Guide for Technical Product Managers

## Introduction

This document serves as a quick reference guide for essential Python libraries, AI tools, and technologies commonly used in modern software production environments. As a Technical Product Manager, understanding these tools enables effective communication with engineering teams, informed decision-making during technical discussions, and better alignment of product strategy with technical capabilities.

The goal is not to provide exhaustive architectural details, but rather to maintain a concise, accessible overview of each technology's purpose, core functionality, and best practices. This guide will help you quickly refresh your knowledge before stakeholder meetings, technical reviews, or product planning sessions.

## Technology Index

| Technology | Description | Category |
|------------|-------------|----------|
| [Sentry](#sentry) | Real-time error tracking and monitoring platform | Monitoring & Observability |
| [OAuth 2.1](#oauth-21) | Industry-standard protocol for authorization | Authentication & Security |
| [Pydantic](#pydantic) | Data validation and settings management using Python type annotations | Data Validation |
| [MCP (Model Context Protocol)](#mcp-model-context-protocol) | Protocol for connecting AI models with external data sources | AI Integration |
| [FastMCP](#fastmcp) | Python framework for rapidly building MCP servers | AI Integration |

---

## Document Structure Guidelines

**Each technology section follows this standard format:**

1. **Technology Name** (as heading)
2. **One-line description** - What the technology does in simple terms
3. **Overview** - 3-4 lines explaining core functionality and use cases
4. **Official Link** - Link to official documentation or website
5. **Best Practices** - 3-5 bullet points covering implementation recommendations
6. **Common Pitfalls** - 2-3 brief points on frequent mistakes
7. **Alternatives** - Brief mention of competing technologies
8. **Decision Criteria** - When to choose this technology
9. **Questions to Ask AI** - 3 questions a TPM should explore when implementing

---

## Technologies

### Sentry

**Real-time error tracking and performance monitoring for production applications**

Sentry is an application monitoring platform that provides real-time error tracking, performance monitoring, and crash reporting across multiple programming languages and frameworks. It automatically captures errors, exceptions, and performance issues in production environments, helping teams identify and resolve problems before they impact users significantly. Sentry offers rich contextual information including stack traces, user impact data, and release tracking.

**Official Link:** [https://sentry.io](https://sentry.io)

**Best Practices:**
- Set up alerts with appropriate thresholds to avoid alert fatigue while catching critical issues
- Use release tracking to correlate errors with specific deployments
- Configure appropriate sampling rates for performance monitoring to manage costs
- Integrate with issue tracking systems (Jira, GitHub) for streamlined workflow
- Implement custom tags and context to enhance error categorization and filtering

**Common Pitfalls:**
- Over-logging verbose errors leading to quota exhaustion and high costs
- Not filtering sensitive data (PII, passwords) from error reports
- Ignoring errors without investigating root causes, defeating the purpose

**Alternatives:**
Datadog, New Relic, Rollbar, LogRocket

**Decision Criteria:**
Choose Sentry when you need specialized error tracking with deep stack traces and release correlation. Opt for alternatives if you require broader infrastructure monitoring or prefer all-in-one APM solutions.

**Questions to Ask AI:**
1. "How should I configure Sentry sampling rates for a Django application handling 1M requests/day to balance cost and visibility?"
2. "What's the recommended Sentry alert configuration strategy to minimize false positives while catching critical production issues?"
3. "How do I implement PII scrubbing in Sentry for a healthcare application to maintain HIPAA compliance?"

---

### OAuth 2.1

**Standardized protocol for secure authorization and delegated access**

OAuth 2.1 is the latest consolidation of the OAuth 2.0 framework, providing a secure and standardized way to grant applications limited access to user accounts on third-party services. It enables "Login with Google/GitHub/etc." functionality and allows applications to access APIs on behalf of users without exposing passwords. OAuth 2.1 incorporates security best practices and simplifies the specification by removing deprecated flows.

**Official Link:** [https://oauth.net/2.1/](https://oauth.net/2.1/)

**Best Practices:**
- Always use PKCE (Proof Key for Code Exchange) for public clients and mobile applications
- Implement proper token storage (never store tokens in localStorage for web apps)
- Use short-lived access tokens with refresh token rotation
- Validate redirect URIs strictly to prevent authorization code interception
- Implement proper scope management to request minimal necessary permissions

**Common Pitfalls:**
- Storing tokens insecurely in browsers or mobile apps
- Not validating state parameters, opening vulnerability to CSRF attacks
- Using implicit flow (deprecated in OAuth 2.1) for single-page applications

**Alternatives:**
OpenID Connect (for authentication), SAML 2.0, Custom JWT-based auth

**Decision Criteria:**
Use OAuth 2.1 for third-party integrations or when building APIs that need delegated access. Choose OpenID Connect when you need authentication identity information. Consider custom auth only for internal-only systems.

**Questions to Ask AI:**
1. "What's the complete OAuth 2.1 implementation checklist for adding Google login to a React/FastAPI application?"
2. "How should I handle token refresh and rotation in a mobile app to maintain security while providing seamless user experience?"
3. "What are the security trade-offs between authorization code flow with PKCE versus client credentials flow for our use case?"

---

### Pydantic

**Data validation and parsing using Python type hints**

Pydantic is a data validation library that leverages Python type annotations to validate, parse, and serialize data structures. It automatically validates incoming data against defined schemas, provides clear error messages for invalid data, and converts data types as needed. Widely used in FastAPI and other modern Python frameworks, Pydantic helps prevent runtime errors by enforcing data contracts at application boundaries.

**Official Link:** [https://docs.pydantic.dev](https://docs.pydantic.dev)

**Best Practices:**
- Define models at API boundaries (request/response validation)
- Use Field validators for complex business logic validation
- Leverage Pydantic's settings management for application configuration
- Enable strict mode for production to catch type coercion issues
- Document models with descriptions for auto-generated API documentation

**Common Pitfalls:**
- Overusing validators for business logic instead of keeping validation focused
- Not handling nested model validation errors properly in API responses
- Confusing Pydantic V1 and V2 syntax when using older documentation

**Alternatives:**
Marshmallow, attrs with validators, dataclasses with manual validation, Cerberus

**Decision Criteria:**
Choose Pydantic for FastAPI projects or when you need runtime validation with type hints. Use Marshmallow if you need more complex serialization logic. Stick with dataclasses for simple data containers without validation needs.

**Questions to Ask AI:**
1. "Show me how to implement custom field validators in Pydantic V2 for validating email domains against an allowed list"
2. "What's the performance impact of Pydantic validation on high-throughput APIs, and how can I optimize it?"
3. "How do I structure Pydantic models for a multi-tenant application where validation rules vary by tenant?"

---

### MCP (Model Context Protocol)

**Open protocol enabling AI assistants to securely connect with external data sources and tools**

MCP is an open-source protocol developed by Anthropic that standardizes how AI models interact with external systems, data sources, and tools. It provides a unified interface for AI assistants to access context from various sources like databases, APIs, file systems, and business tools. MCP enables building custom integrations that extend AI capabilities while maintaining security boundaries and proper access controls.

**Official Link:** [https://modelcontextprotocol.io](https://modelcontextprotocol.io)

**Best Practices:**
- Design MCP servers around specific, well-defined use cases (database access, file operations, etc.)
- Implement proper authentication and authorization for sensitive data access
- Use resource-based access patterns to expose data hierarchically
- Document available tools and resources clearly for AI assistant discovery
- Monitor and log MCP server usage for security and debugging

**Common Pitfalls:**
- Exposing too much functionality in a single MCP server, creating security risks
- Not implementing proper error handling, causing cryptic AI responses
- Forgetting to version MCP server APIs, breaking integrations on updates

**Alternatives:**
Custom API endpoints, LangChain tools, Function calling with direct API integration, OpenAI plugins (deprecated)

**Decision Criteria:**
Use MCP when building AI assistants that need standardized access to multiple data sources. Choose direct function calling for simpler, single-purpose integrations. Consider LangChain for complex agent workflows with existing ecosystem support.

**Questions to Ask AI:**
1. "What's the recommended architecture for an MCP server that provides AI access to our PostgreSQL database with row-level security?"
2. "How do I implement rate limiting and access controls in an MCP server to prevent AI from overwhelming backend systems?"
3. "What's the best practice for handling sensitive data in MCP responses while maintaining useful context for the AI?"

---

### FastMCP

**Python framework for rapidly building MCP servers**

FastMCP is a high-level Python framework that simplifies the development of MCP servers with minimal boilerplate code. Built on top of the official MCP SDK, it provides decorators and utilities for quickly exposing Python functions as MCP tools and resources. FastMCP handles protocol complexity, connection management, and serialization automatically, allowing developers to focus on business logic rather than protocol implementation details.

**Official Link:** [https://github.com/jlowin/fastmcp](https://github.com/jlowin/fastmcp)

**Best Practices:**
- Use type hints for automatic parameter validation and documentation
- Structure servers around domain-specific functionality (one server per integration)
- Leverage async functions for I/O-bound operations
- Implement proper error handling to return meaningful messages to AI assistants
- Test MCP servers with MCP Inspector tool before deployment

**Common Pitfalls:**
- Not using async/await for I/O operations, blocking the event loop
- Over-abstracting tools when simpler implementations would suffice
- Ignoring type hints, losing automatic validation benefits

**Alternatives:**
Raw MCP SDK, custom protocol implementation, LangChain tool wrappers

**Decision Criteria:**
Use FastMCP for rapid prototyping and production MCP servers in Python. Use raw MCP SDK when you need fine-grained control or language-specific features. Consider alternatives if your team isn't Python-focused.

**Questions to Ask AI:**
1. "Show me a complete FastMCP server implementation that exposes our company's customer data API with proper authentication"
2. "How do I deploy and scale FastMCP servers in a Kubernetes environment with multiple AI assistant clients?"
3. "What's the testing strategy for FastMCP servers to ensure reliability when used by AI assistants in production?"

---

## Adding New Technologies

When adding new entries to this document, maintain the structure above and keep each section concise. Focus on actionable information that helps with product decisions rather than implementation details. The "Questions to Ask AI" section should reflect real scenarios a TPM might encounter when evaluating or implementing the technology.