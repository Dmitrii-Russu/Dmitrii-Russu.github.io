---
title: "Where Domain Roles Should Live: The Boundary Between the Authorization Server and the Resource Server"
date: 2026-09-28 08:00:00 +0300
categories: [Architecture, Backend]
tags: [spring, spring-security, oauth2, jwt, authorization-server, resource-server]
---

In short: domain roles belong where the domain lives. Putting them on the auth server couples it to someone else's business model, and baking them into the JWT freezes them for the lifetime of the token. Mapping the token's identity to a domain user on the resource server keeps the boundary clean and puts you in control of how fast a revoked role takes effect.

Your resource server needs a `userId` and domain roles. Your JWT doesn't carry them. So where should they live?

It's a question every team building on OAuth2 and Spring eventually faces, and the answer quietly shapes your architecture: who owns the domain model, how fast a revoked role takes effect, and how tightly your auth server is coupled to someone else's business logic.

Below are three common approaches, what each one really costs, and why I believe the boundary between the Authorization Server and the Resource Server should stay where it belongs.

A resource server often needs more than what a JWT carries: an internal `userId`, domain roles, domain-specific attributes. There are three ways to get them, and each has its price.

## Approach #1: A Custom User on the Authorization Server

The most direct path is to extend the user model: add a UUID instead of a username key, your own roles, your own fields.

Spring Security ships with a ready-made `JdbcUserDetailsManager` for the standard users/authorities schema. If your domain model requires a fundamentally different user structure, or a different way of representing it in `UserDetails`, the standard JDBC flow needs extra configuration, up to a custom `UserDetailsService`/`AuthenticationProvider`. That is now a piece of the auth server that **you** write and maintain, not the Spring team.

**There is also a less obvious cost.** With JDBC persistence, the attributes of `OAuth2Authorization` are serialized to JSON. Depending on the authorization flow, those attributes may contain the authenticated `Principal`. If it turns out to be a custom principal type, the standard Jackson mixins in Spring Authorization Server won't cover it. They are built for the standard `Authentication`/`Principal` types. Without additional serialization setup, restoring a stored authorization fails with a deserialization error.

> **If the custom User carries a resource server's domain-specific ID and roles, the auth server starts depending on that server's domain model, which it was never supposed to do.**

## Approach #2: Domain Claims via `OAuth2TokenCustomizer`

The alternative is to leave the user model alone and enrich the token itself: add `userId` and roles directly to the JWT at issuance.

A customizer can indeed add arbitrary claims. The question is **where those claims come from**. If the auth server has to reach into the resource server's domain data, it creates a coupling that wasn't there before. Either the auth server stores its own copy of that data, or it fetches it synchronously at issuance, putting the resource server on the critical path of login.

**A subtler problem is data freshness.** An issued token never changes, no matter what happens to the user's role in the database. New claims appear only at the next issuance, for example on refresh. The access token TTL bounds how fresh the information *inside the token* is, and in this approach the domain roles live exactly there. So as long as the token is alive, a revoked role is simply not reflected in it.

**Scaling makes it worse.** As the number of resource servers with different role sets grows, either the token carries the union of all roles, or the auth server starts deciding which claims are meant for which audience, gradually turning into an orchestrator of someone else's authorization.

This doesn't mean claims in tokens are a bad idea in themselves. For coarse, cross-domain roles (organization-level admin/user/guest), it's an established pattern. Keycloak realm roles and Auth0 Actions work this way.

> **The problem appears with fine-grained domain roles of a specific resource server: the more granular and specific they are, the more this path ties the auth server to business logic that doesn't belong to it.**

## Approach #3: Identity Mapping on the Resource Server

The third path leaves the auth server untouched. The token doesn't carry a domain-specific user model. Interpreting the subject in a domain context happens where that domain lives: **on the resource server**.

Here, the freshness of domain roles is determined not by the token TTL but by the resource server's own lookup and caching policy. But for the lookup to work at all, you need a reliable key that tells the resource server whose role it is. And there is a non-obvious detail here.

### Why the Key Is a Pair, Not a Single Value

In the default Spring Authorization Server configuration, `sub` often matches the username, but you shouldn't rely on that.

- For an OIDC ID Token, the stability of `iss` + `sub` is guaranteed by the protocol itself: `sub` must be locally unique and never reassigned within an issuer.
- For a JWT access token, which is what a resource server usually works with, `sub` is defined by the contract of the specific access token profile. For example, RFC 9068 requires `sub` and states that, for grants involving a resource owner, it SHOULD correspond to that resource owner's subject identifier.

So the resource server should rely on the contract of the specific Authorization Server, not on the mere assertion "this is a JWT."

With multiple issuers, the same `sub` can theoretically occur at different ones.

> **That's why the mapping key on a resource server must be the pair `(iss, sub)`, not a bare `sub`.**

Having received a valid token, the resource server extracts `(iss, sub)`, finds its internal `userId` by it, then the domain roles and attributes by that `userId`. These end up in the security context alongside the token's own data.

### A Separate Question: Who Is the Token For?

`(iss, sub)` identifies the subject. But there is a separate question, "who was this token issued for?", and it is answered by `aud`. RFC 9068 explicitly ties this field to the target resource.

A resource server must validate both the issuer and the audience of a token. In Spring Security, `aud` validation requires explicit configuration: the basic setup via `issuer-uri` covers `iss`/`exp`/`nbf`, but not `aud`.

> **If `aud` doesn't match the current resource server, the token must be rejected before identity mapping.** Otherwise the subject may be valid and known to the mapping, yet the token was still issued for a different resource.

## Where This Fits in Spring

The entry point is a `Converter<Jwt, AbstractAuthenticationToken>`, plugged in via `JwtAuthenticationConverter`. This is the standard extension point: `sub` is used as the default principal claim, and the converter exists precisely to replace the logic of building an `Authentication` from a token.

**Where you resolve roles determines how the rest of the security system behaves, and the two options are not equivalent.**

- **Resolving in the converter.** Domain roles land in the `Authentication` immediately, as ordinary `GrantedAuthority` objects. Further down the filter chain, `hasRole(...)` and `@PreAuthorize` work with no additional changes. For them, it's just another authority.
- **Deferring resolution until the authorization decision.** The converter returns a "bare" `Authentication`, with only what is actually in the token. But `hasRole(...)` and `@PreAuthorize` read roles from the `Authentication`; they don't fetch them separately. So with deferred resolution, the standard mechanism simply won't see them. You'll need either a custom permission-check implementation or a separate method-security mechanism that performs the lookup as well.

> **These are not two interchangeable ways to achieve the same thing. They are different integration mechanics with the authorization system, and you need to choose deliberately.**

## What Else to Consider

**Stateless applies to the bearer token, not the whole application.** A JWT-based resource server doesn't keep server-side state about the access token between requests. But after identity mapping, it does keep its own state: the `(iss, sub)` → `userId` mapping and a role cache.

**Caching and freshness.** Without a cache, the identity-mapping lookup runs on every request to a protected endpoint. That's perfectly workable under low load, or when freshness matters more than performance. Under significant load, the lookup may need caching, and then its TTL becomes a tunable freshness threshold:

- A **local cache** is simple, but with multiple instances a role change on one of them isn't immediately reflected in the caches of the others.
- A **distributed cache** removes that divergence, but doesn't remove the invalidation question: you still need either an explicit change signal or the same TTL as the source of freshness.

**This is the key difference from domain claims in the token.**

> **With identity mapping, a revoked role is visible on the very next request (no cache), or once the cache TTL expires. And that TTL is controlled by the resource server, independently of the token's lifetime. With domain claims, that isn't possible: as long as the token lives, the revoked role stays visible until it expires.**

## Conclusion

Of the three approaches, **the third best preserves separation of concerns**: the Authorization Server is responsible for authentication and token issuance, while the Resource Server interprets identity in the context of its own domain.

- **Custom User** is itself a standard extension mechanism of the Authorization Server. But putting a specific resource server's domain `userId` and roles there couples the auth layer to someone else's business model. With excessive customization, the Authorization Server can end up reproducing the functions of a full IAM platform instead of using a specialized tool for the job.
- **Domain claims** are also a standard mechanism, and they fit when the permissions are shared across several resources and can live with access token semantics. But for specific, frequently changing roles of a particular resource server, identity mapping decouples the lifecycle of domain permissions from the token TTL.
- **When you need full centralized management** of a complex role model, access policies, per-client visibility and an admin UI, evaluate a specialized IAM platform such as Keycloak, rather than gradually rebuilding its functions on top of Spring Authorization Server.
