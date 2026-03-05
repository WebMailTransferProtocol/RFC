# WMTP – Web Mail Transfer Protocol (RFC Draft)

WMTP treats email transport as a web-native capability rather than a specialized server role. If you can deploy a web application, you can deploy a WMTP endpoint — no dedicated mail daemon, no separate ports, no parallel infrastructure to operate.

Messages transmitted using WMTP are called **wmail** (pronounced /ˈwiːmeɪl/, like "we-mail," where "we" stands for web). A wmail is a mail message transported natively over HTTPS according to the WMTP specification.

## Purpose of This RFC

This RFC defines the WMTP protocol semantics, message format and transport rules, server and client responsibilities, error handling, interoperability constraints, and security considerations. It targets developers implementing WMTP servers, client library authors, hosting providers, infrastructure engineers, and researchers exploring alternative mail transport paradigms.

## Rationale

WMTP originates from a simple observation: most environments already run a web server. Introducing a separate mail server adds operational complexity without proportional benefit.

### Stack Simplification and Developer Accessibility

A dedicated SMTP daemon brings its own configuration surface, ports, security model, and monitoring requirements. WMTP eliminates that layer entirely. A single hardened web server handles both application traffic and mail transport. HTTP is universally understood — libraries, tooling, and debugging workflows are already built around it — so implementing a WMTP client or server requires no specialized mail protocol expertise.

### Scalability and Network Compatibility

Because WMTP operates over HTTPS, it integrates directly with existing reverse proxies, load balancers, and geographic distribution infrastructure. Standard HTTP-based scaling strategies apply without modification. WMTP avoids non-standard ports, which means it works naturally within restrictive firewall environments and cloud networking policies where SMTP is often blocked.

### Deployability

Anyone who can publish a web application can provision a WMTP mailbox. Shared hosting environments, VPS instances, container platforms, serverless HTTP runtimes, and free hosting providers all qualify. A functional WMTP server can be running in minutes. This removes the traditional constraints of mail server provisioning, DNS complexity, and provider lock-in.

### Observability

Because WMTP traffic is HTTPS-based, it inherits existing logging infrastructure, metrics collection, distributed tracing, API gateway integration, and rate limiting — immediately, without additional configuration. Operational insight is consistent with every other web service in the stack.

### Security Alignment

Because WMTP runs over HTTPS, TLS encryption is inherited by default — no additional configuration required. Certificate management, authentication middleware, and web security best practices are already mature within HTTP ecosystems. WMTP reuses these mechanisms rather than redefining them.

## Design Philosophy

WMTP favors minimalism over feature accretion, explicit behavior over implicit heuristics, transport clarity over historical compatibility, and practical deployability over theoretical completeness. The protocol does not attempt to replicate all historical SMTP behaviors. It defines a constrained, web-native message transfer model.

## Repository Structure

The protocol specification is in `RFC.md`. This document is the entry point; `RFC.md` is the authoritative technical reference.

## Status

This is an early draft RFC, open to revision. The protocol has not yet reached a stable release. Feedback, implementation reports, and protocol critiques are welcome.

## Contributing

Open an issue to propose changes, report ambiguities, or discuss interoperability concerns. Pull requests are accepted for clarifications to the specification, improvements to security considerations, and reference implementations. All proposals should preserve the core philosophy of infrastructure simplification and web-native transport.

## License

This project is licensed under the BSD 3-Clause License. See the `LICENSE` file for the full text.
