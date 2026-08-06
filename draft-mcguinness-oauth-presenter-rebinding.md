---
title: "Presenter Rebinding for OAuth 2.0 Proof-of-Possession Tokens"
abbrev: "OAuth Presenter Rebinding"
category: std

docname: draft-mcguinness-oauth-presenter-rebinding-latest
submissiontype: IETF
number:
date:
v: 3
ipr: "trust200902"
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - oauth
 - proof of possession
 - dpop
 - key binding
 - presenter rebinding
 - token exchange
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "mcguinness/draft-mcguinness-oauth-presenter-rebinding"
  latest: "https://mcguinness.github.io/draft-mcguinness-oauth-presenter-rebinding/draft-mcguinness-oauth-presenter-rebinding.html"

author:
 -
    fullname: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

normative:
  RFC6749:
  RFC6838:
  RFC7515:
  RFC7517:
  RFC7519:
  RFC7638:
  RFC7800:
  RFC8414:
  RFC8693:
  RFC8707:
  RFC8725:
  RFC9449:

informative:
  RFC9068:
  RFC9700:
  I-D.mcguinness-oauth-cross-client-delegation:
    title: "Cross-Client Delegation Profile for OAuth 2.0 Token Exchange"
    author:
     -
        fullname: Karl McGuinness
        organization: Independent
    date: 2026
    target: https://datatracker.ietf.org/doc/html/draft-mcguinness-oauth-cross-client-delegation
  I-D.mcguinness-oauth-actor-profile:
    title: "OAuth Actor Profile for Delegation"
    author:
     -
        fullname: Karl McGuinness
        organization: Independent
    date: 2026-04-30
    target: https://datatracker.ietf.org/doc/html/draft-mcguinness-oauth-actor-profile-00
  OpenID.KeyBinding:
    title: "OpenID Connect Key Binding 1.0 - draft 02"
    author:
     -
        fullname: Dick Hardt
        organization: Hellō
     -
        fullname: Ethan Heilman
        organization: Cloudflare
    date: 2026-06-24
    target: https://openid.net/specs/openid-connect-key-binding-1_0.html

...

--- abstract

A proof-of-possession (PoP) token can normally be presented only by a party that holds the token's confirmation key.  This prevents an intended recipient from presenting the token when a token handoff is authorized but transferring the confirmation private key is unacceptable.  This document defines a one-hop presenter-rebinding mechanism for OAuth 2.0 Token Exchange.  The holder of a JWT Source Token's confirmation key signs a Presenter Rebinding Assertion (PRA) that authorizes a recipient key to present that exact token at a named authorization server.  The recipient presents the Source Token and PRA with a DPoP proof of its own key.  After applying normal authorization policy, the authorization server can issue a new token bound to the recipient key.  Further presenter changes use another exchange rather than an offline delegation chain.


--- middle

# Introduction

Proof-of-possession (PoP) binding transforms a bearer security token into one that only its intended holder can use.  A JWT can carry a confirmation (`cnf`) claim {{RFC7800}} identifying a key, and the presenter proves possession of that key when using the token.  OpenID Connect Key Binding {{OpenID.KeyBinding}} applies this model to ID Tokens, and JWT access tokens {{RFC9068}} and Token Exchange {{RFC8693}} outputs can carry `cnf` as well.

PoP binding creates a gap whenever the party that should present a token is not the party the token is bound to.  This arises in more than one place:

*  In cross-client delegation {{I-D.mcguinness-oauth-cross-client-delegation}}, a Delegate presents an Identity Assertion issued to and key-bound to an Initiator.  The Delegate does not hold the Initiator's key.

*  In presenter-transition and rebind flows under the OAuth Actor Profile {{I-D.mcguinness-oauth-actor-profile}}, a token bound to one presenter has to be continued by another.

*  More generally, any Token Exchange in which the requesting party differs from the party a key-bound `subject_token` is bound to faces the same gap, whatever profile governs the exchange.

Two unsatisfactory options are common today.  Removing the binding turns the token into a replayable bearer credential and discards the protection PoP binding provides.  Sharing the confirmation private key defeats key isolation and is often impossible.  This document defines a third option that requires neither.

This document defines a one-hop cryptographic authorization for changing the presenter.  The holder of the Source Token's confirmation key signs a **Presenter Rebinding Assertion (PRA)** that binds together the exact Source Token, a recipient key, and one authorization server.  The recipient submits the Source Token and PRA in a Token Exchange request and proves possession of the recipient key with DPoP {{RFC9449}}.  The authorization server validates both signatures and applies all ordinary Token Exchange, client, actor, and deployment policy before issuing a token bound to the recipient key.

The PRA answers one question: did the Source Token's confirmation-key holder authorize this recipient key to present this token at this authorization server?  It does not identify the party controlling either key, grant scopes or resources, or establish that an actor is authorized to act for a subject.  Those decisions remain with the applicable OAuth profile and authorization-server policy.

Presenter rebinding is deliberately one hop.  If the recipient later needs to authorize another presenter, it first obtains a new token bound to its own key and creates a new PRA for that token.  This keeps each proof path constant in size and ensures that the authorization server evaluates policy at every presenter transition.  Offline, recursively signed delegation chains are out of scope.

## Requirements and Scope {#scope}

This specification applies to a Source Token that:

*  is a JWT whose exact encoded value is available to both the confirmation-key holder and the authorization server;
*  is accepted as a `subject_token` in OAuth 2.0 Token Exchange; and
*  carries an RFC 7800 `cnf` claim that resolves to exactly one asymmetric signing key as specified in {{key-identification}}.

The recipient proves possession with DPoP, and the recipient key is the key proven by the request's single DPoP proof, as specified in {{te-request}}.  Symmetric confirmation keys, bearer Source Tokens, non-JWT Source Tokens, mutual-TLS presenter rebinding, use outside Token Exchange, and offline multi-hop delegation are out of scope.  A future specification can define another profile without changing the one-hop security invariant defined here.

This document changes nothing for direct presentation.  A party that holds the Source Token's confirmation key presents the token under its existing profile and does not use a PRA.

Presenter rebinding is opt-in.  An authorization server MUST accept a PRA only under an applicable profile and local policy that explicitly permit presenter rebinding for the Source Token type and request context.

The following are also out of scope:

*  how the confirmation-key holder decides to authorize the handoff, including user interaction or consent;
*  how the confirmation-key holder authenticates the intended recipient key, except for the requirements in {{recipient-key}};
*  establishing trust in the Source Token issuer; and
*  the policy for authenticating or authorizing the clients, actors, subject, requested resources, or requested scopes.


# Conventions and Definitions {#conventions}

{::boilerplate bcp14-tagged}

Unless otherwise specified, OAuth and JOSE terms are used as defined by {{RFC6749}}, {{RFC7515}}, {{RFC7517}}, {{RFC7519}}, {{RFC7800}}, {{RFC8693}}, and {{RFC9449}}.

Source Token:
: A JWT used as the Token Exchange `subject_token`, carrying an RFC 7800 `cnf` claim, whose presenter is being rebound.

Source Confirmation Key:
: The asymmetric signing key identified by the Source Token's `cnf` claim.

Original Presenter:
: The party that holds the Source Confirmation Key and signs a PRA.  The term describes key control and does not, by itself, identify an OAuth client or actor.

Recipient Presenter:
: The party that controls the key identified by a PRA `cnf` claim, submits the Token Exchange request, and proves possession of that key with DPoP.

Presenter Rebinding Assertion (PRA):
: A signed JWT by which the Original Presenter authorizes a Recipient Presenter key to present one exact Source Token at one authorization server.

Presenter Limits:
: Optional upper bounds in a PRA on the Token Exchange target audiences, resources, and scopes that can result from the request.  Presenter Limits restrict a request but never grant authority.


# Protocol Model {#model}

Presenter rebinding has three cryptographically distinct inputs:

1. The Source Token establishes the Source Confirmation Key.
2. The PRA, signed by that key, authorizes the Recipient Presenter key for the exact Source Token and authorization server.
3. The DPoP proof demonstrates that the Token Exchange requester controls the Recipient Presenter key and binds that proof to the token-endpoint request.

All three inputs are required.  A PRA without the Recipient Presenter's DPoP proof can be copied but not used.  That DPoP proof without a PRA does not satisfy the Source Token's confirmation requirement.  Neither artifact replaces validation of the Source Token or authorization of the Token Exchange request.

One key serves as both the Recipient Presenter key and the key the requester proves on the request, and the token the exchange issues is bound to that key; {{te-request}} and {{te-processing}} state the requirements.  A later presenter transition starts from that newly issued token and requires a new one-hop PRA.  The authorization server, rather than a recursively nested assertion, carries forward any authorized actor history in the issued token according to the applicable actor profile.


# Presenter Rebinding Assertion {#pra}

A Presenter Rebinding Assertion is a JWT {{RFC7519}} secured as a JWS {{RFC7515}}.

## JOSE Header {#pra-header}

The protected JOSE header MUST contain:

*  `typ`: the explicit type `pra+jwt` ({{iana-media-type}}).  As permitted by {{RFC7515, Section 4.1.9}}, the `application/` prefix is omitted.  An authorization server MUST reject a JWT presented as a PRA whose `typ` is not `pra+jwt`.

*  `alg`: an asymmetric digital-signature algorithm accepted by the authorization server and appropriate for the key in `jwk`.  An authorization server MUST reject `none`, symmetric algorithms, algorithms outside its allowlist, and an algorithm inconsistent with the key type.  An authorization server advertises the algorithms it accepts as specified in {{metadata}}.

*  `jwk`: the public Source Confirmation Key, represented as an asymmetric JWK {{RFC7517}} containing only public parameters.  An authorization server MUST reject a `jwk` containing private-key material.

The `jwk` is not trusted merely because it appears in the header.  Its JWK SHA-256 Thumbprint MUST match the key identifier derived from the validated Source Token as specified in {{validation}}.

An authorization server MUST resolve the PRA signing key only from the `jwk` header parameter.  It MUST NOT dereference a `jku` or `x5u` header parameter and MUST NOT resolve the signing key from `x5c` or `kid`.  Dereferencing a presenter-supplied URI exposes the authorization server to request forgery and serves no purpose here, because the signing key is fully determined by the validated Source Token's `cnf` claim.

## Claims {#pra-claims}

A PRA MUST contain:

*  `sth` (Source Token hash): the base64url encoding without padding of the SHA-256 digest of the ASCII octets of the exact encoded Source Token value.  This is the construction used by the DPoP `ath` claim ({{RFC9449, Section 4.2}}), applied to the Source Token.  The value is computed before transport encoding.  For Token Exchange, it is computed over the `subject_token` value before `application/x-www-form-urlencoded` encoding, and the authorization server computes it after form decoding.  It MUST NOT be computed over decoded claims, reserialized JSON, or a canonicalized representation.

*  `cnf`: the Recipient Presenter key.  It MUST be a JSON object containing a single member, `jkt`, whose value is the JWK SHA-256 Thumbprint {{RFC7638}} of the Recipient Presenter's asymmetric DPoP public key.  An authorization server MUST reject a PRA whose `cnf` contains any other member.  This specification defines only `jkt`, because DPoP is the only proof mechanism it defines; an extension introducing another confirmation method MUST specify the corresponding proof mechanism and MUST preserve the requirement that `cnf` identify exactly one asymmetric key ({{extensibility}}).

*  `aud`: the authorization server issuer identifier.  Its value MUST be one case-sensitive string, not an array.  The value MUST exactly equal the issuer identifier by which the authorization server identifies itself.  It is distinct from the Token Exchange `audience` request parameter and the DPoP `htu` claim.

*  `iat`: the time at which the PRA was issued.

*  `exp`: the expiration time.  A PRA SHOULD be short-lived, and an authorization server MUST enforce a maximum PRA lifetime according to local policy.

A PRA MAY contain:

*  `jti`: an identifier for audit, profile-defined one-time use, or profile-defined revocation.  Reuse of a PRA `jti` is not by itself a replay because a PRA authorizes a key and can be used more than once during its validity interval unless a profile says otherwise.

*  `presenter_limits`: a JSON object restricting the Token Exchange result as specified in {{limits}}.  Its defined members are `audience`, `resource`, and `scope`.  An authorization server MUST reject a member it does not recognize; recognized members are those registered per {{iana-limits}} and supported by the authorization server.

Other claims MUST NOT be interpreted as identifying the Original Presenter or granting authority unless an applicable profile explicitly defines that meaning and its validation rules.

## Presenter Limits {#limits}

`presenter_limits` is an optional restriction on the result of the one Token Exchange request.  It is an upper bound, not a grant.  Successful PRA validation never overrides the Source Token's authorization rules, an administered relationship, a `may_act` claim, client or actor authentication, user authorization, or authorization-server policy.

The object can contain:

*  `audience`: an array of case-sensitive Token Exchange audience strings.  A requested or issued audience is within this limit when it is a member of the array.

*  `resource`: an array of resource indicator URI strings {{RFC8707}}.  Each member MUST be an absolute URI without a fragment component, as required by {{RFC8707, Section 2}}.  A requested or issued resource is within this limit when it equals a member of the array by simple string comparison.  This specification defines no prefix, path-hierarchy, or wildcard relationship between resource indicators.  An authorization server that cannot determine containment by string comparison MUST reject the request.

*  `scope`: a space-delimited set of scope values.  A requested or granted scope value is within this limit when it is a member of the set.

When a member is absent, the PRA places no restriction on that dimension.

When a member is present, the corresponding Token Exchange request parameter MUST be present in the request, and every value it carries MUST be within the limit.  An authorization server MUST reject a request that omits a parameter for which the PRA carries a limit.  Requiring the requester to state what it wants removes the case in which an authorization-server default, rather than the request, determines the result for a limited dimension.  A Recipient Presenter holds the PRA and can therefore read each limit it must request within.

The authorization server MAY issue a result narrower than both the request and the Presenter Limits.  It MUST NOT issue a result broader than either.

Together these rules confine the issued token by construction: the request is explicit for every limited dimension, the request is within the limit, and the issued authorization is within the request.  An authorization server whose issuance cannot produce a value outside the request for a limited dimension therefore satisfies this section by comparing the request against the limit, at the point where it already validates `audience`, `resource`, and `scope`.  {{security-limits}} states the requirement that remains on the issued authorization.

This specification does not define limits for `authorization_details`; a profile that needs such limits must define type-specific containment rules in an extension and register the member per {{iana-limits}}.  A profile whose base already carries `authorization_details` through Token Exchange therefore loses an upper bound on that dimension when it composes with this document, and needs such an extension to restore it.  {{security-limits}} states the requirements an extension must meet.

## Confirmation-Key Identification {#key-identification}

The authorization server reduces the validated Source Token's `cnf` claim to one expected JWK SHA-256 Thumbprint.  A `cnf` containing `jkt` supplies that value directly.  A `cnf` containing a public asymmetric `jwk` supplies the value obtained by applying {{RFC7638}} with SHA-256.

The authorization server MUST reject presenter rebinding when the Source Token's `cnf` identifies a symmetric key, identifies more than one key, cannot be reduced to exactly one asymmetric public signing key or SHA-256 thumbprint, or uses an unsupported confirmation method.  The authorization server MUST NOT resolve the Source Confirmation Key from untrusted material supplied by the Recipient Presenter except for a public `jwk` whose thumbprint is compared with a trusted `jkt` from the validated Source Token.

## Example {#pra-example}

The following abbreviated PRA authorizes `K_recipient` to present one Source Token at `https://idp.example` and limits the exchange result to one target audience.

~~~json
{
  "typ": "pra+jwt",
  "alg": "ES256",
  "jwk": { "kty": "EC", "crv": "P-256", "x": "...", "y": "..." }
}
~~~
{: title="PRA protected header"}

~~~json
{
  "sth": "0Ei...source-token-hash...",
  "cnf": { "jkt": "0ZcOCORZNYy-DWpqq30jZyJGHTN0d2HglBV3uiguA4I" },
  "aud": "https://idp.example",
  "presenter_limits": {
    "audience": ["https://gateway.example"]
  },
  "iat": 1785974400,
  "exp": 1785974700,
  "jti": "b1f0c8a2-9d3e-4a2b-9f1c-8e7d6c5b4a30"
}
~~~
{: title="PRA claims"}


# Presentation and Validation {#validation}

The Recipient Presenter sends the Source Token as the Token Exchange `subject_token`, the compact PRA in the `presenter_rebinding` parameter, and a DPoP proof in the `DPoP` HTTP header on the same request.

The authorization server MUST perform the following checks.  Failure of any required check means that presenter rebinding is not authorized.  Every check MUST be performed against a single request: the Source Token, PRA, DPoP proof, and Token Exchange parameters MUST all be taken from the same HTTP request, and the authorization server MUST NOT combine artifacts drawn from different requests.

1. Confirm that an applicable profile and local policy permit presenter rebinding for the Source Token type and request context.

2. Validate the Source Token according to its token type and profile, including its cryptographic protection, issuer, audience, temporal validity, and every check not explicitly replaced by the applicable profile.  Reduce its `cnf` to exactly one expected Source Confirmation Key thumbprint per {{key-identification}}.

3. Parse the PRA without accepting duplicate JSON member names.  Confirm that the protected header contains `typ`, `alg`, and a public `jwk` as required by {{pra-header}}.  Compute the header key's JWK SHA-256 Thumbprint and confirm that it equals the expected Source Confirmation Key thumbprint.  Verify the PRA signature with that header key and accepted `alg`.

4. Confirm that `sth` equals the value defined in {{pra-claims}}, computed over the `subject_token` parameter after form decoding, that `aud` exactly identifies the authorization server, and that `iat` and `exp` satisfy temporal-validity, clock-skew, and maximum-lifetime policy.  Apply `jti` replay or revocation state only when required by an applicable profile or local policy.

5. Confirm that the PRA `cnf` contains only `jkt` and that its value is syntactically valid.  Validate the DPoP proof carried on this request according to {{RFC9449}}, including its signature, `typ`, `alg`, `jwk`, `jti`, `htm`, `htu`, `iat`, and nonce when required.  Confirm that the DPoP public key's JWK SHA-256 Thumbprint exactly equals the PRA `cnf.jkt`.

6. Validate the Token Exchange request, client and actor authentication, delegation relationship, and requested authorization under {{RFC8693}}, the applicable profile, and local policy.  Confirm that the request carries a parameter for every dimension the PRA limits, and apply Presenter Limits per {{limits}} to those request values and to the resulting authorization.

On success, the Source Token's proof-of-possession requirement for this Token Exchange presentation is satisfied by the holder of the PRA Recipient Presenter key.  This result establishes control of an authorized presentation key.  It does not establish the presenter's application-level identity or independently authorize the exchange.


# Use with OAuth 2.0 Token Exchange {#token-exchange}

## Request {#te-request}

This specification defines the following Token Exchange parameter:

`presenter_rebinding`:
: The compact JWS serialization of one PRA rooted in the `subject_token`'s confirmation key.  It is REQUIRED when the requester uses this mechanism and MUST NOT occur more than once.

The `subject_token` is the Source Token.  The PRA `aud` MUST be the authorization server's issuer identifier; it is not the token endpoint URI, the DPoP `htu`, or the requested token audience.

The requester MUST send a DPoP proof on the same request, and the key proven by that DPoP proof MUST be the key identified by the PRA `cnf.jkt`.  {{RFC9449}} permits at most one DPoP proof per request, and at the token endpoint that proof also determines the key to which the authorization server sender-constrains the issued token.  A requester using this mechanism therefore uses the Recipient Presenter key as its DPoP key for this request; it cannot present one key as the Recipient Presenter and a different key for sender-constraining.  An authorization server MUST reject a request whose DPoP proof is for a key other than the PRA `cnf.jkt`.

This constrains only key control on this request.  It does not merge the Recipient Presenter and OAuth client roles, and client authentication remains an independent input per {{security-separation}}.

## Processing and Output {#te-processing}

The authorization server validates the request according to {{validation}}.  If it issues a token, the authorization carried by that token MUST be within both normal authorization policy and any Presenter Limits.

The authorization server MUST sender-constrain the issued token to the PRA Recipient Presenter key.  Because that key is also the key proven by the request's DPoP proof ({{te-request}}), this is the binding ordinary DPoP token-endpoint processing already produces.  Sender-constraining completes the presenter transition: the issued token has one current confirmation key, and a subsequent transition requires a new PRA signed by that key over the new token.

An authorization server MUST NOT issue a bearer token in response to a request that uses presenter rebinding.  Doing so would convert a key-bound Source Token into an unbound credential and defeat both the Source Token's confirmation requirement and the one-hop invariant in {{security-transitions}}.  If the authorization server is unwilling to sender-constrain the requested token type, it MUST reject the request rather than issue an unbound token.

An applicable delegation profile MAY allow the authorization server to record the Original Presenter as a prior actor in the issued token's `act` claim {{RFC8693}}.  It can do so only when the profile independently maps the Source Confirmation Key to an authenticated actor identity and authorizes that actor relationship.  A key thumbprint is not an actor identity, and a valid PRA alone is insufficient to create an `act` entry.

## Errors {#te-errors}

Errors are returned according to {{RFC8693, Section 2.2.2}} and {{RFC6749, Section 5.2}}.  When presenter rebinding is required, each of the following uses the `invalid_request` error code:

*  a missing or invalid `presenter_rebinding` parameter, or more than one occurrence of it;
*  PRA validation failure under {{validation}};
*  DPoP validation failure;
*  a DPoP proof for a key other than the PRA `cnf.jkt` ({{te-request}});
*  a request that omits a Token Exchange parameter for which the PRA carries a limit ({{limits}});
*  a request or resulting authorization outside Presenter Limits; and
*  an inability to sender-constrain the requested token type ({{te-processing}}).

An unacceptable `audience` or `resource` SHOULD instead use `invalid_target`, as specified by {{RFC8693}}.  DPoP-specific error processing follows {{RFC9449}}.  Error descriptions SHOULD NOT disclose which key, relationship, or policy input caused rejection.

## Authorization Server Metadata {#metadata}

An authorization server advertises support in its metadata {{RFC8414}} with the following OPTIONAL members:

`presenter_rebinding_supported`:
: Boolean value indicating support for the `presenter_rebinding` Token Exchange parameter and the validation rules in this document.  The value is `true` when supported.  If omitted, support is not indicated.  This metadata does not guarantee that any particular Source Token, algorithm, client, actor, or request will be accepted.

`presenter_rebinding_signing_alg_values_supported`:
: JSON array containing a list of the JWS `alg` values supported by the authorization server for the PRA signature.  The array MUST NOT contain `none` or a symmetric algorithm.  An authorization server that supports presenter rebinding SHOULD include this member, because {{pra-header}} requires it to reject algorithms outside its allowlist and {{te-errors}} discourages disclosing the reason for rejection.  If omitted, an Original Presenter has no discovery mechanism for the accepted algorithms and must obtain them out of band.


# Use in Delegation Profiles {#profiles}

A delegation profile can use a valid PRA as evidence that the Source Confirmation Key authorized a specific key handoff.  It MUST continue to authenticate and authorize the parties and relationship independently.

A profile using this mechanism MUST define each of the following.  A profile that leaves any of them undefined has not specified a complete composition with this document:

1. Which Source Tokens may be presented with a PRA, and in which request contexts, as the opt-in required by {{scope}}.

2. How the Original Presenter obtains and authenticates the intended Recipient Presenter key, or which deployment mechanism supplies that process, as required by {{recipient-key}}.

3. How the authorization server maps the Source Confirmation Key and the Recipient Presenter key to party identities, where those identities affect authorization or actor recording.

4. How the PRA combines with the profile's other authorization inputs, including client and actor authentication, administered relationships, `may_act`, and exchange-time policy.

5. Whether the profile authorizes recording the Original Presenter as a prior actor, and if so, the `act` depth and cycle limits required by {{security-transitions}}.

A profile MUST NOT treat the PRA or Presenter Limits as a grant of authority.

The Cross-Client Delegation profile {{I-D.mcguinness-oauth-cross-client-delegation}} composes with this document.  The following sketch is illustrative only; that profile's own composition section is the normative binding between the two documents, and it governs where the two differ.

*  the Source Token is the Initiator's key-bound Identity Assertion;
*  the Initiator controls the Source Confirmation Key and signs the PRA;
*  the Delegate controls the Recipient Presenter key, submits the exchange, and proves possession with DPoP;
*  the IdP is the authorization server; and
*  the PRA is conjunctive with the administered cross-client relationship, `may_act` when present, Delegate client and actor authentication, and exchange-time policy.


# Extensibility {#extensibility}

This document defines one artifact, one proof mechanism, and one binding of that artifact to a protocol exchange.  Each is an extension point.  This section states what an extension may add and what it MUST NOT change, so that additional profiles and use cases can compose without renegotiating the security model.

An extension MAY:

*  define an additional `presenter_limits` member, subject to {{iana-limits}} and to the containment requirements in {{security-limits}}.  Registration is what keeps independent extensions from choosing the same member name: an unrecognized member already fails closed under {{pra-claims}}, but without a registry a collision is not detectable at the time either extension is written;

*  define an additional PRA confirmation method for `cnf`, together with the proof mechanism by which a Recipient Presenter demonstrates control of the identified key; or

*  define an additional binding of the PRA to a protocol exchange other than the Token Exchange binding in {{token-exchange}}.  Such a binding MUST specify the value that `aud` carries for that exchange, how the PRA is transported, how the Recipient Presenter proves possession on the same request, and how the result of a successful presentation is constrained.  {{scope}} places use outside Token Exchange outside the scope of this document; it does not reserve it against a specification that supplies those definitions.

An extension MUST preserve each of the following invariants.  An extension that changes any of them is a different protocol and requires its own validation, revocation, privacy, and resource-exhaustion analysis.  Each invariant is stated normatively in the section referenced beside it; the list below summarizes those requirements and does not restate them, and the referenced section governs.

1. **One hop.**  A PRA authorizes one presenter transition, and is never nested, chained, or read as authorizing another PRA ({{security-transitions}}).

2. **One authorization server.**  A PRA names exactly one authorization server in `aud`, as one case-sensitive string, and is valid only there ({{pra-claims}}).  A binding to another protocol exchange defines what `aud` carries for that exchange, but names exactly one verifying party.

3. **Exact token binding.**  A PRA authorizes presentation of the one Source Token whose encoded octets `sth` covers, and of no other token ({{pra-claims}}).

4. **No presenter-chosen signing key.**  The PRA signing key is determined by the validated Source Token's `cnf` claim, and a Recipient Presenter cannot influence which key verifies the PRA ({{key-identification}}).

5. **One Recipient Presenter key, proven on the request.**  A PRA identifies exactly one asymmetric Recipient Presenter key, and control of that key is proven on the same request that carries the PRA ({{te-request}}).

6. **Limits narrow only.**  Presenter Limits and any extension to them are upper bounds; they never grant authority, and their omission is never permission ({{security-limits}}).


# Security Considerations {#security}

The OAuth 2.0 Security Best Current Practice {{RFC9700}} and JWT Best Current Practices {{RFC8725}} apply in addition to this section.

## Explicit Acceptance of Rebinding {#security-opt-in}

Key binding ordinarily assures a verifier that the presenter controls the key selected when the token was issued.  Presenter rebinding deliberately permits a different key to satisfy that requirement for one Token Exchange presentation.  This is an authorization-semantic change, not a generic consequence of RFC 7800.  Authorization servers MUST opt in for each applicable Source Token profile and request context and MUST continue to enforce issuer and deployment policy.

## Separation of Key Control, Identity, and Authority {#security-separation}

The Source Token authenticates its issuer and establishes the Source Confirmation Key.  The PRA proves authorization from that key to the Recipient Presenter key.  DPoP proves control of the Recipient Presenter key on the request.  None of those facts alone identifies the party controlling a key or authorizes an OAuth delegation relationship.  Client authentication, actor authentication, administered relationships, `may_act`, consent, and authorization-server policy remain independent and conjunctive inputs.

## Recipient Key Authentication {#recipient-key}

Before signing a PRA, the Original Presenter MUST obtain the intended Recipient Presenter's JWK or thumbprint through an authenticated, integrity-protected process providing the assurance required by the application.  If an attacker substitutes its own key before signature, the resulting PRA validly authorizes that attacker key.  A consuming profile MUST define the key-distribution and authentication process or identify the deployment mechanism that supplies it.

## Replay and Freshness {#security-replay}

A captured Source Token and PRA cannot be used without control of the Recipient Presenter key.  Binding the PRA to the exact Source Token, one authorization server, one Recipient Presenter key, and a short validity interval limits replay.  DPoP provides per-request replay protection through its `jti`, `iat`, request target, method, and nonce when used.

A PRA is not one-time-use by default.  A deployment that requires one-time use needs atomic replay-cache processing and retry semantics keyed by `jti`; such a deployment requires `jti` by policy.  Offline revocation of a PRA is unavailable unless a profile defines a lookup mechanism, so short lifetimes are the primary control.

This has an aggregate consequence that Original Presenters need to account for.  Presenter Limits bound the result of one Token Exchange request, not the total authorization obtainable from one PRA.  Until the PRA expires, the Recipient Presenter can repeat the exchange and obtain a separate token for each audience, resource, and scope combination the limits and policy allow.  A PRA limited to three audiences authorizes up to three tokens, not one.

Presenter Limits cannot express a single-use restriction, and no authorization server check derives one from them.  A deployment that needs exactly one exchange requires the profile-defined one-time-use processing described above.  Absent that, an Original Presenter SHOULD keep the PRA lifetime close to the time the Recipient Presenter needs to make its request.

## Request Binding {#security-request-binding}

PRA `aud` and DPoP request binding provide different protections.  `aud` expresses authorization for one authorization server.  DPoP `htu` and `htm` bind proof of the Recipient Presenter key to one HTTP target and method.  Both MUST be checked.

DPoP does not cover the HTTP request body.  The authorization server MUST process the Source Token, PRA, and Token Exchange parameters from the same TLS-protected request on which it validates DPoP.  Presenter Limits cryptographically express upper bounds selected by the Original Presenter, but normal TLS and request processing remain necessary to prevent parameter mixing.

## Token and Key Substitution {#security-substitution}

The `sth` claim prevents a PRA created for one Source Token from being combined with another.  Matching the PRA header `jwk` thumbprint to the validated Source Token `cnf` prevents a Recipient Presenter from choosing the PRA signing key.  Matching PRA `cnf.jkt` to the DPoP key prevents substitution of the current presenter proof.

Original Presenters SHOULD use confirmation keys dedicated to key-bound tokens.  Signing APIs MUST bind the `pra+jwt` type and intended operation so that an attacker cannot use a generic signing oracle to obtain a PRA.

## Presenter Limits Are Not Grants {#security-limits}

Presenter Limits only narrow authorization.  An authorization server MUST apply all other authorization inputs independently and MUST NOT treat omission of a limit as permission.

A present limit is an input to the computation that produces the granted authorization, alongside the request, the Source Token's authorization, and local policy.  It is not a check appended after issuance.  This is why {{limits}} rejects a request that omits a parameter for a limited dimension rather than defaulting it: a default computed without the limit as an input can exceed it, and a server that discovers this only by inspecting the finished token has already done the work twice.  The authorization placed in the issued token MUST NOT exceed a present limit, whatever other input would otherwise have produced a broader result.

An authorization server that supports only some registered `presenter_limits` members fails closed rather than open.  {{pra-claims}} requires it to reject a PRA carrying a member it does not support, so a limit an authorization server cannot enforce is never silently ignored.

This specification omits generic `authorization_details` containment because arbitrary authorization-detail types do not share a safe comparison operation.  Extensions adding such limits must define type-specific semantics and fail closed when containment cannot be determined.

## Subsequent Presenter Transitions {#security-transitions}

The Recipient Presenter cannot extend a PRA or sign a PRA for the original Source Token because it does not hold the Source Confirmation Key.  A later transition requires a successful exchange yielding a new token bound to the current presenter's key.  This forces the authorization server to reevaluate policy at every transition and keeps the PRA proof path constant in size, avoiding recursive parsing, cycle handling, and offline propagation of authority.

The proof path is bounded; the actor history is not.  Each transition that a profile authorizes can add one `act` entry to the issued token per {{te-processing}}, so repeated presenter transitions grow that nesting one level at a time even though no PRA is ever nested.  This specification does not bound `act` depth, detect cycles among prior actors, or define disclosure rules for actor history.  A profile that authorizes actor recording MUST specify those limits; see also the OAuth Actor Profile for Delegation {{I-D.mcguinness-oauth-actor-profile}}.

Implementations MUST NOT accept a nested PRA or interpret a PRA as authorizing another PRA.  A specification that defines offline multi-hop delegation is a different protocol and requires separate validation, revocation, privacy, and resource-exhaustion analysis.

## Downgrade and Confusion {#security-downgrade}

An authorization server MUST NOT accept bearer presentation of a key-bound Source Token because a PRA is absent or invalid.  The Source Token requires either direct proof of its confirmation key under its own profile or a valid PRA and a matching Recipient Presenter DPoP proof under this specification.

The output side requires the same discipline.  Issuing a bearer token from a successful presenter-rebinding exchange downgrades the Source Token's key binding one step later: the Recipient Presenter obtains a replayable credential derived from a token that could only ever be presented with a key.  {{te-processing}} therefore requires the issued token to be sender-constrained to the Recipient Presenter key and requires the authorization server to reject the request rather than issue an unbound token.

The explicit `pra+jwt` type and mutually exclusive validation rules distinguish a PRA from a Source Token, access token, ID Token, client assertion, and DPoP proof.  An authorization server MUST validate each artifact only under the rules for its protocol position.

## Trust Domains {#security-domains}

The authorization server can validate a PRA only if it can validate the Source Token and obtain its confirmation-key identifier without trusting presenter-supplied key resolution.  Across trust domains, the authorization server must trust the Source Token issuer, token profile, and confirmation method.  Establishing that trust is outside this specification.


# Privacy Considerations {#privacy}

A PRA discloses the Source Confirmation public key and Recipient Presenter key thumbprint to the authorization server.  These values can become correlation handles.  Parties SHOULD use keys with the narrowest practical lifetime and scope and SHOULD avoid reusing a Recipient Presenter key across unrelated relationships when correlation is a concern.

Unlike an offline delegation chain, a PRA does not disclose intermediate key paths.  Actor history is included in an issued token only when required and authorized by an applicable profile, allowing that profile and issuer to apply its disclosure policy.


# IANA Considerations {#iana}

## Media Type Registration {#iana-media-type}

This document requests registration of the following media type in the "Media Types" registry, following the procedures of {{RFC6838}}, for use as the `typ` JOSE header value of a Presenter Rebinding Assertion.  As permitted by {{RFC7515, Section 4.1.9}}, the `application/` prefix is omitted in the `typ` value.

*  Type name: application
*  Subtype name: pra+jwt
*  Required parameters: N/A
*  Optional parameters: N/A
*  Encoding considerations: binary; a PRA is a JWT represented as base64url-encoded values separated by periods
*  Security considerations: See {{security}} and Section 11 of {{RFC7519}}
*  Interoperability considerations: N/A
*  Published specification: this document
*  Applications that use this media type: OAuth applications rebinding presentation of proof-of-possession tokens
*  Fragment identifier considerations: N/A
*  Additional information: Magic number(s): N/A; File extension(s): N/A; Macintosh file type code(s): N/A
*  Person and email address to contact for further information: Karl McGuinness, public@karlmcguinness.com
*  Intended usage: COMMON
*  Restrictions on usage: N/A
*  Author: Karl McGuinness
*  Change controller: IETF
*  Provisional registration: No

## JSON Web Token Claims Registration {#iana-claims}

This document requests registration of the following claims in the "JSON Web Token Claims" registry established by {{RFC7519}}.  The `cnf`, `aud`, `iat`, `exp`, and `jti` claims are already registered and are used without new registration.

*  Claim Name: `sth`
*  Claim Description: Source Token Hash; base64url-encoded SHA-256 digest of the token whose presenter is being rebound
*  Change Controller: IESG
*  Specification Document: {{pra-claims}} of this document

and:

*  Claim Name: `presenter_limits`
*  Claim Description: Upper bounds on authorization resulting from a presenter-rebound Token Exchange
*  Change Controller: IESG
*  Specification Document: {{pra-claims}} and {{limits}} of this document

## Presenter Limits Members Registry {#iana-limits}

This document requests creation of a new registry, "Presenter Limits Members", to hold the members defined for the `presenter_limits` claim ({{limits}}).

The registration procedure is Specification Required.  The designated expert is directed to confirm that a proposed member specifies a containment rule that compares a requested value and the corresponding authorization in the issued token, that the rule can be evaluated by simple comparison or is otherwise fully specified, and that it fails closed when containment cannot be determined, as required by {{security-limits}}.  A member that can broaden the result of an exchange MUST NOT be registered.

Each registration contains a Member Name, a Description, a Change Controller, and a Specification Document.  The initial contents are the three members defined by this document:

*  Member Name: `audience`; Description: Upper bound on Token Exchange target audiences; Change Controller: IESG; Specification Document: {{limits}} of this document

*  Member Name: `resource`; Description: Upper bound on resource indicators; Change Controller: IESG; Specification Document: {{limits}} of this document

*  Member Name: `scope`; Description: Upper bound on granted scope values; Change Controller: IESG; Specification Document: {{limits}} of this document

## OAuth Parameters Registration {#iana-parameter}

This document requests registration of the following value in the "OAuth Parameters" registry established by {{RFC6749}}.

*  Parameter name: `presenter_rebinding`
*  Parameter usage location: token request
*  Change Controller: IESG
*  Specification Document: {{te-request}} of this document

## OAuth Authorization Server Metadata Registration {#iana-metadata}

This document requests registration of the following values in the "OAuth Authorization Server Metadata" registry established by {{RFC8414}}.

*  Metadata Name: `presenter_rebinding_supported`
*  Metadata Description: Boolean indicating authorization server support for presenter rebinding in OAuth 2.0 Token Exchange
*  Change Controller: IESG
*  Specification Document: {{metadata}} of this document

and:

*  Metadata Name: `presenter_rebinding_signing_alg_values_supported`
*  Metadata Description: JSON array of JWS `alg` values supported by the authorization server for Presenter Rebinding Assertion signatures
*  Change Controller: IESG
*  Specification Document: {{metadata}} of this document


--- back

# Worked Example: Cross-Client Delegation {#appendix-example}

This informative example shows presenter rebinding used with the Cross-Client Delegation profile {{I-D.mcguinness-oauth-cross-client-delegation}} so that a Delegate can present a key-bound ID Token issued to an Initiator.

The Initiator holds an ID Token bound to `K_init`.  The Delegate conveys the public JWK or thumbprint for `K_del` to the Initiator over the authenticated, integrity-protected mechanism defined by the deployment profile.  The Initiator verifies that the key belongs to the intended Delegate and signs a PRA with `K_init`.  The PRA binds the exact ID Token to `K_del`, names the IdP as the authorization server, limits the result to `https://gateway.example`, and expires after five minutes.  The Initiator conveys the exact encoded ID Token and PRA to the Delegate.

The Delegate performs Token Exchange at the IdP:

~~~
POST /token HTTP/1.1
Host: idp.example
Content-Type: application/x-www-form-urlencoded
DPoP: <proof of possession of K_del>

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&subject_token=<key-bound Initiator ID Token>
&subject_token_type=urn:ietf:params:oauth:token-type:id_token
&actor_token=<Delegate actor token>
&actor_token_type=urn:ietf:params:oauth:token-type:jwt
&audience=https://gateway.example
&presenter_rebinding=<PRA signed by K_init for K_del>
&client_assertion_type=<jwt-bearer-client-assertion-type>
&client_assertion=<Delegate client assertion>
~~~

The IdP validates the ID Token under the composed profile, reduces its `cnf` to `jkt(K_init)`, confirms that the PRA is signed by `K_init`, verifies the Source Token hash and IdP audience, and verifies the Delegate's DPoP proof of `K_del`.  Because the PRA carries an `audience` limit, the request must carry an `audience` parameter, and the IdP checks the requested value against that limit rather than defaulting it.  It separately authenticates the Delegate as client and actor, validates the administered cross-client relationship and `may_act` when present, and applies exchange-time policy.

The single `K_del` proof serves both roles required by {{te-request}}: it satisfies the PRA `cnf.jkt` and it sender-constrains the token the IdP issues.  Note that the Delegate authenticates as a client with a separate client assertion, so client authentication remains independent of key control.

On success, the IdP issues a token sender-constrained to `K_del`.  The applicable profile can authorize it to record the Delegate as the current actor and the Initiator as a prior actor:

~~~json
{
  "iss": "https://idp.example",
  "sub": "<user identifier for the gateway>",
  "aud": "https://gateway.example",
  "cnf": { "jkt": "0ZcOCORZNYy-DWpqq30jZyJGHTN0d2HglBV3uiguA4I" },
  "act": {
    "iss": "https://idp.example",
    "sub": "delegate-client",
    "act": {
      "iss": "https://idp.example",
      "sub": "initiator-client"
    }
  }
}
~~~

An attacker that captures the Initiator's ID Token and PRA cannot perform the exchange without `K_del`.  If the Delegate later hands authority to another presenter, it uses the newly issued `K_del`-bound token as the Source Token in a new exchange; it does not extend the original PRA.


# Acknowledgments
{:numbered="false"}

This mechanism addresses the presenter-transition needs of OAuth delegation profiles while keeping key handoff separate from actor identity and authorization policy.  It is designed to compose with OpenID Connect Key Binding {{OpenID.KeyBinding}}, DPoP {{RFC9449}}, OAuth 2.0 Token Exchange {{RFC8693}}, the OAuth Actor Profile {{I-D.mcguinness-oauth-actor-profile}}, and the Cross-Client Delegation profile {{I-D.mcguinness-oauth-cross-client-delegation}}.


# Document History
{:numbered="false"}

\[\[ To be removed from the final specification ]]

-00

* Initial revision.
