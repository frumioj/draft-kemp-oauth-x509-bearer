---
title: "X.509 Certificate Bearer Profile for OAuth 2.0 Client Authentication and Authorization Grants"
abbrev: "OAuth X.509 Bearer"
category: std

docname: draft-kemp-oauth-x509-bearer-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - oauth
 - x.509
 - pki
 - client authentication
 - assertion
 - bearer token
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "frumioj/draft-kemp-oauth-x509-bearer"
  latest: "https://frumioj.github.io/draft-kemp-oauth-x509-bearer/draft-kemp-oauth-x509-bearer.html"

author:
 -
    fullname: "John Kemp"
    organization: Your Organization Here
    email: "unstable.pseudonym@stabledomain.net"

normative:
  RFC5280:
  RFC6749:
  RFC7521:
  RFC6755:

informative:
  RFC7522:
  RFC7523:
  RFC8705:
  RFC6960:
  RFC8446:
  RFC6973:
  SPIFFE-X509-SVID:
    title: "SPIFFE X.509 SVID"
    target: "https://github.com/spiffe/spiffe/blob/main/standards/X509-SVID.md"
    date: false
    author:
      -
        org: "SPIFFE Project"
  ATHENZ-X509:
    title: "Athenz: Using X.509 Certificates"
    target: "https://athenz.github.io/athenz/service_x509_credentials/"
    date: false
    author:
      -
        org: "Athenz Project"

...

--- abstract

This specification defines the use of an X.509 certificate, issued under a
Public Key Infrastructure (PKI), as a means for requesting an OAuth 2.0
access token as well as for client authentication, profiling the Assertion
Framework for OAuth 2.0 Client Authentication and Authorization Grants
({{RFC7521}}) in a manner analogous to the JSON Web Token (JWT) Bearer
Token profile ({{RFC7523}}) and the SAML 2.0 Bearer Assertion profile
({{RFC7522}}). It is motivated primarily by workload identity systems,
such as SPIFFE/SPIRE and Athenz, that already issue software workloads
short-lived X.509 certificates for mutual TLS, and that benefit from
using those same certificates directly with OAuth 2.0. Unlike a bare
bearer credential, this profile requires that possession of the private
key corresponding to the certificate's public key be corroborated as part
of every use, so that a copy of the certificate alone -- which is not a
secret -- is never sufficient to obtain a grant or authenticate a client.


--- middle

# Introduction

JSON Web Token (JWT) {{RFC7523}} and Security Assertion Markup Language
(SAML) 2.0 {{RFC7522}} bearer assertion profiles allow a client that
already holds a security token, issued by a trusted party, to present that
token directly to an OAuth 2.0 {{RFC6749}} authorization server (AS) --
either as an authorization grant or as a client authentication mechanism
-- without minting a purpose-built, single-use assertion for every
request. This specification is motivated primarily by workload identity
systems: infrastructure that automatically issues and rotates short-lived
X.509 certificates {{RFC5280}} to software workloads -- services,
processes, or automated agents, as distinct from human end users -- as
their means of cryptographic identity, typically as part of a service
mesh or zero-trust network architecture. SPIFFE and its SPIRE runtime
{{SPIFFE-X509-SVID}} and Athenz {{ATHENZ-X509}} are two examples of such
systems in current production use; both issue workloads an X.509
certificate carrying a URI Subject Alternative Name (SAN), commonly using
the "spiffe://" URI scheme, as the workload's primary identifier (see
{{terminology}}). A workload that already holds such a certificate, and
that already uses it to authenticate over mutual TLS to its peers,
benefits from being able to use that same certificate and trust
relationship directly with an OAuth 2.0 authorization server, instead of
provisioning and operating a separate JWT issuer solely to obtain OAuth
tokens. This specification is not limited to workload identity
deployments, but they are its primary motivating use case, and are
referenced throughout as running examples.

This specification profiles the OAuth Assertion Framework {{RFC7521}} to
define an extension grant type that uses an X.509 certificate to request
an OAuth 2.0 access token, as well as for use as a client authentication
credential. The format and processing rules defined here are
intentionally similar, though not identical, to those in {{RFC7523}} and
{{RFC7522}}; the differences arise from the structure and semantics of
X.509 certificates, which were designed to bind a public key to a subject
identity for use across many contexts and over comparatively long
lifetimes, and not to express single-transaction claims such as an
intended audience or an issuance time for one specific request.

This specification is related to, but distinct from, "OAuth 2.0 Mutual-TLS
Client Authentication and Certificate-Bound Access Tokens" {{RFC8705}}.
RFC 8705 authenticates a client using the certificate presented during
the TLS handshake with the authorization server itself, and does not
define a way to use a certificate as an authorization grant. This
specification instead carries a reference to (or, when necessary, a copy
of) an X.509 certificate as an assertion value within the token request,
which allows the same underlying PKI-issued identity to also be used as
an authorization grant, and accommodates deployments in which TLS is
terminated in front of the authorization server's application logic (for
example, by a load balancer or reverse proxy) and the verified client
certificate is forwarded to the token endpoint by a trusted intermediary.
Because a certificate, unlike a private key, is not itself a secret,
Section 3 of this document requires that proof of possession of the
associated private key be corroborated by one of these mechanisms as part
of processing every request; see also Section 6 (Security
Considerations). Once a grant has been obtained or a client
authenticated under this profile, {{cnf}} recommends carrying that same
proof-of-possession property forward into the issued access token by
reusing RFC 8705's certificate-bound access token mechanism, so that PKI
possession is required consistently across the whole exchange rather than
only at the token endpoint.

The process by which the client obtains its X.509 certificate, prior to
using it with the authorization server, is out of scope of this document.

## Notational Conventions

{::boilerplate bcp14-tagged}

Unless otherwise noted, all the protocol parameter names and values are
case sensitive.

## Terminology {#terminology}

All terms are as defined in the following specifications: "The OAuth 2.0
Authorization Framework" {{RFC6749}}, the OAuth Assertion Framework
{{RFC7521}}, and "Internet X.509 Public Key Infrastructure Certificate
and Certificate Revocation List (CRL) Profile" {{RFC5280}}.

X.509 Bearer Certificate:
: An X.509 certificate, valid under {{RFC5280}}, used as an assertion
  per this specification, together with a corroborated proof that the
  presenting party possesses the private key corresponding to the
  certificate's public key.

Issuing CA:
: The Certification Authority identified in the Issuer field of an X.509
  Bearer Certificate.

Workload:
: A software workload -- a service, process, or automated agent -- as
  distinct from a human end user, that acts as an OAuth client, a
  resource owner, or both, in its own right.

Workload Identity Certificate:
: An X.509 Bearer Certificate issued to a Workload by a workload identity
  control plane (for example, a SPIRE server implementing SPIFFE
  {{SPIFFE-X509-SVID}}, or Athenz {{ATHENZ-X509}}), as distinct from a
  certificate issued to a human end user by a general-purpose enterprise
  or public PKI. Workload Identity Certificates are typically valid for
  a short, fixed lifetime, are rotated automatically and frequently by
  the issuing control plane well before expiry, and identify the
  Workload using a URI SubjectAltName, commonly one using the
  "spiffe://" URI scheme, rather than (or in addition to) a Subject
  distinguished name.

# HTTP Parameter Bindings for Transporting X.509 Certificate Assertions

The OAuth Assertion Framework {{RFC7521}} defines generic HTTP parameters
for transporting assertions (a.k.a. security tokens) during interactions
with a token endpoint. This section defines specific parameters and
treatments of those parameters for use with X.509 Bearer Certificates.

## Using X.509 Certificates as Authorization Grants {#grants}

To use an X.509 Bearer Certificate as an authorization grant, the client
uses an access token request as defined in Section 4 of the OAuth
Assertion Framework {{RFC7521}} with the following specific parameter
values and encodings.

The value of the "grant_type" is
"urn:ietf:params:oauth:grant-type:x509-bearer".

The value of the "assertion" parameter MUST contain a single X.509
certificate, encoded as the base64url {{RFC7521}}-style encoding
(unpadded, per the base64url alphabet) of the certificate's DER
{{RFC5280}} encoding, or, when the client reaches the token endpoint over
a mutually authenticated TLS connection, a thumbprint reference to that
same certificate as defined in {{thumbprint}}.

The "scope" parameter may be used, as defined in the OAuth Assertion
Framework {{RFC7521}}, to indicate the requested scope.

Authentication of the client is optional, as described in Section 3.2.1
of OAuth 2.0 {{RFC6749}}, and consequently the "client_id" is only needed
when a form of client authentication that relies on the parameter is
used.

The following example demonstrates an access token request with an X.509
certificate as an authorization grant (with extra line breaks for display
purposes only):

~~~
POST /token.oauth2 HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ax509-bearer
&assertion=MIIDXTCCAkWgAwIBAgIJAKrX1234abcd
[...omitted for brevity...]
~~~

## Using X.509 Certificates for Client Authentication {#client-auth}

To use an X.509 Bearer Certificate for client authentication, the client
uses the following parameter values and encodings.

The value of the "client_assertion_type" is
"urn:ietf:params:oauth:client-assertion-type:x509-bearer".

The value of the "client_assertion" parameter contains a single X.509
certificate, encoded as described in {{grants}}, or a thumbprint
reference as described in {{thumbprint}}. It MUST NOT contain more than
one certificate or reference.

The following example demonstrates client authentication using an X.509
certificate during the presentation of an authorization code grant in an
access token request (with extra line breaks for display purposes only):

~~~
POST /token.oauth2 HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=n0esc3NRze7LTCu7iYzS6a5acc3f0ogp4&
client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3A
client-assertion-type%3Ax509-bearer&
client_assertion=MIIDXTCCAkWgAwIBAgIJAKrX1234abcd
[...omitted for brevity...]
~~~

## Referencing the Certificate Presented for Mutual TLS {#thumbprint}

When the client reaches the token endpoint over a mutually authenticated
TLS {{RFC8446}} connection, the certificate it presented during that TLS
handshake is already available to the authorization server, and
embedding a full copy of it a second time in the "assertion" or
"client_assertion" value is unnecessary. In this case, the client MAY
instead send a reference to that certificate, consisting of the literal
string "x5t#S256:" followed immediately by the base64url-encoded
(unpadded) SHA-256 digest of the certificate's DER {{RFC5280}} encoding
-- computed identically to the "x5t#S256" confirmation method value
defined in Section 3.1 of {{RFC8705}}. Because neither "#" nor ":" occur
in the base64url alphabet, this reference form can never be mistaken for
the base64url-encoded certificate encoding described in {{grants}} and
{{client-auth}}.

An authorization server that receives an "assertion" or
"client_assertion" value in this form MUST independently obtain the
actual certificate from a source it trusts for the corroboration
required by item 7 of {{format}} -- either the certificate presented on
its own TLS connection with the client, or a certificate or thumbprint
forwarded by a trusted intermediary -- and MUST verify that the SHA-256
digest of that certificate's DER encoding matches the referenced
thumbprint before proceeding. An authorization server MUST NOT accept a
thumbprint reference together with a certificate supplied elsewhere in
the client's request as if the two were mutually corroborating; the
certificate the thumbprint is checked against MUST come from the TLS
layer or a trusted intermediary, never from the request itself.

This reference form is only meaningful when proof of possession is
otherwise corroborated per item 7 of {{format}}; it is a shorthand for
avoiding retransmission of a certificate the authorization server already
has, not a substitute for that corroboration.

The following example shows the client authentication example above
expressed as a thumbprint reference instead of a full certificate, using
the certificate from the example in {{grants}}, with extra line breaks
for display purposes only:

~~~
POST /token.oauth2 HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=n0esc3NRze7LTCu7iYzS6a5acc3f0ogp4&
client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3A
client-assertion-type%3Ax509-bearer&
client_assertion=x5t%23S256%3AkOpPSFiF3aL3tqsXOu72sHNitAB75QQLs6R81Q_yX6g
~~~

# X.509 Certificate Format and Processing Requirements {#format}

In order to issue an access token response as described in OAuth 2.0
{{RFC6749}} or to rely on an X.509 certificate for client authentication,
the authorization server MUST validate the certificate and the request
according to the criteria below. Throughout this section, "the
certificate" refers to the X.509 certificate conveyed by the "assertion"
or "client_assertion" value, whether encoded directly as described in
{{grants}} and {{client-auth}} or identified by a thumbprint reference as
described in {{thumbprint}}. Application of additional restrictions and
policy are at the discretion of the authorization server.

1. The certificate MUST be well-formed and valid under {{RFC5280}},
   MUST chain to a trust anchor configured at the authorization server
   (either directly or via a configured set of intermediate CA
   certificates), and MUST NOT be revoked, as determined per the
   authorization server's configured revocation-checking mechanism (for
   example, CRLs or OCSP {{RFC6960}}).

2. The Issuer field of the certificate identifies the Issuing CA, which
   fills a role equivalent to the "iss" claim in {{RFC7523}}. The
   authorization server MUST have a pre-established trust relationship
   with the Issuing CA, and MUST NOT accept certificates from CAs outside
   its configured set of trust anchors.

3. The Subject field and/or the subjectAltName extension of the
   certificate identifies the principal that is the subject of the
   certificate, filling a role equivalent to the "sub" claim in
   {{RFC7523}}. Two cases need to be differentiated:

   A. For the authorization grant, the subject typically identifies an
      authorized accessor for which the access token is being requested
      (i.e., the resource owner or an authorized delegate).

   B. For client authentication, the subject, or a specific
      subjectAltName value designated for this purpose by prior
      agreement between the client and the authorization server, MUST
      correspond to the "client_id" of the OAuth client.

   The authorization server MUST be configured, per Issuing CA, with the
   rule used to extract the relevant identifier from the Subject field or
   subjectAltName extension. For a Workload Identity Certificate
   ({{terminology}}), this identifier is typically a URI SAN, commonly
   one using the "spiffe://" URI scheme as issued by SPIFFE/SPIRE
   {{SPIFFE-X509-SVID}} or Athenz {{ATHENZ-X509}}; authorization servers
   supporting Workload Identity Certificates SHOULD extract the subject
   identifier from the URI SAN in preference to the Subject field, which
   such certificates often leave empty or unpopulated.

4. X.509 certificates as defined in {{RFC5280}} have no field with
   semantics equivalent to the JWT "aud" claim. Audience restriction for
   this profile is therefore established out of band between the
   Issuing CA (or its operator) and the authorization server, as part of
   the same agreement that establishes the trust anchor relationship
   described in item 2 above; see also {{interop}}. Where the subject
   identifier is a "spiffe://" URI SAN as described in item 3, the trust
   domain component of that URI (the authority component immediately
   following the scheme) MAY additionally be used by the authorization
   server as an informal, coarse-grained signal for which relying
   parties a given Issuing CA's certificates are intended for, but doing
   so is a matter of local policy and does not by itself substitute for
   the out-of-band trust agreement described above. A future revision of
   this specification, or a companion specification, MAY define an X.509
   certificate extension carrying an explicit, in-band audience
   restriction for deployments that require it; in the absence of such an
   extension, the authorization server MUST rely on its out-of-band
   configuration to determine whether it is an intended relying party for
   certificates issued by a given Issuing CA.

5. The certificate's notBefore and notAfter validity fields fill roles
   equivalent to the "nbf" and "exp" claims in {{RFC7523}}. The
   authorization server MUST reject any certificate that is not currently
   within its validity period, subject to allowable clock skew between
   systems. The authorization server MAY reject certificates whose
   remaining validity period is unreasonably long for use as a bearer
   assertion under this profile; see {{interop}}.

6. The combination of the Issuer field and the certificate's serialNumber
   fills a role equivalent to the "jti" claim in {{RFC7523}} as a unique
   identifier for the token. The authorization server MAY track
   (Issuer, serialNumber) pairs it has already processed, within a
   configured window, to detect and reject replayed requests; see
   {{security}}.

7. The authorization server MUST corroborate that the party presenting
   the certificate possesses the private key corresponding to the
   certificate's public key. This corroboration MUST be established by
   one of the following means:

   A. The certificate presented as the "assertion" or "client_assertion"
      value is the same certificate the client presented during mutual
      TLS authentication of the connection to the token endpoint, per
      {{RFC8446}}, thereby directly proving possession as part of the TLS
      handshake; or

   B. The certificate was forwarded to the authorization server by a
      trusted intermediary (for example, a TLS-terminating reverse proxy
      or load balancer situated directly in front of the authorization
      server) over a channel the authorization server trusts, together
      with an attestation from that intermediary that the certificate was
      successfully used to authenticate a TLS client during the
      connection now being forwarded.

   A certificate presented as an "assertion" or "client_assertion" value
   without either form of corroboration MUST be rejected. See
   {{security}}. When the "assertion" or "client_assertion" value is a
   thumbprint reference per {{thumbprint}}, mechanism (A) is satisfied
   directly: the authorization server computes the comparison thumbprint
   from the certificate it obtained via its own TLS handshake with the
   client, rather than from a certificate encoded in the request itself.

8. The authorization server MUST reject a certificate that is not valid
   in all other respects per {{RFC5280}}, including but not limited to
   key usage and extended key usage constraints applicable to this use,
   where such constraints are configured by the authorization server.

## Authorization Grant Processing

X.509 Bearer Certificate authorization grants may be used with or without
client authentication or identification. Whether or not client
authentication is needed in conjunction with an X.509 Bearer Certificate
authorization grant, as well as the supported types of client
authentication, are policy decisions at the discretion of the
authorization server. However, if client credentials are present in the
request, the authorization server MUST validate them.

If the certificate is not valid, or the current time is not within the
certificate's valid time window for use, or proof of possession cannot be
corroborated as required by item 7 of {{format}}, the authorization
server constructs an error response as defined in OAuth 2.0 {{RFC6749}}.
The value of the "error" parameter MUST be the "invalid_grant" error
code. The authorization server MAY include additional information
regarding the reasons the certificate was considered invalid using the
"error_description" or "error_uri" parameters.

For example:

~~~
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store

{
 "error":"invalid_grant",
 "error_description":"Certificate chain did not validate to a
  trusted root"
}
~~~

## Client Authentication Processing

If the client's certificate is not valid, the authorization server
constructs an error response as defined in OAuth 2.0 {{RFC6749}}. The
value of the "error" parameter MUST be the "invalid_client" error code.
The authorization server MAY include additional information regarding
the reasons the certificate was considered invalid using the
"error_description" or "error_uri" parameters.

## Certificate-Bound Access Tokens {#cnf}

When an authorization server issues an access token as a result of
successfully processing an X.509 Bearer Certificate authorization grant
({{grants}}), or after authenticating a client using an X.509 Bearer
Certificate under this profile ({{client-auth}}), the authorization
server SHOULD bind the resulting access token to that certificate using
the certificate-bound access token mechanism defined in Section 3 of
{{RFC8705}}, rather than issuing a plain bearer access token. Doing so
carries the proof-of-possession property established when the
certificate was corroborated per item 7 of {{format}} forward to the
resource server, using the same "x5t#S256" confirmation method value
already reused for the reference form defined in {{thumbprint}}, so that
use of the access token also requires possession of the same private key
that was required to obtain it.

An authorization server MAY instead issue a plain bearer access token
{{?RFC6750}} when local policy determines that certificate binding is
unnecessary for the resource being protected, but doing so discards the
proof-of-possession property this profile otherwise provides throughout
the request, and SHOULD be a deliberate policy choice rather than a
default.

# Authorization Grant Example

The following example illustrates what a conforming certificate and an
access token request would look like.

The example shows a Workload Identity Certificate ({{terminology}})
issued to a "billing" workload by the Issuing CA identified as
"CN=Example Workload CA, O=Example Corp, C=US". The subject of the
certificate is identified by a "spiffe://" URI SAN,
"spiffe://example.org/ns/prod/sa/billing", per {{SPIFFE-X509-SVID}}, and
is valid for one hour, consistent with automatic rotation by the issuing
workload identity control plane. The authorization server
"https://authz.example.net" has been configured, out of band, to trust
certificates issued by this CA, and to treat the trust domain
"example.org" as identifying an intended relying party, as described in
{{interop}}. The client presents the certificate over a mutually
authenticated TLS connection to the authorization server's token endpoint
at "https://authz.example.net/token.oauth2".

Below is a summary of the relevant fields of the certificate:

~~~
Issuer: CN=Example Workload CA, O=Example Corp, C=US
Subject: (empty)
Subject Alternative Name: URI = spiffe://example.org/ns/prod/sa/billing
Validity:
    Not Before: 2026-08-19T12:00:00Z
    Not After:  2026-08-19T13:00:00Z
Serial Number: 4096
~~~

To present this certificate as part of an access token request, the
client -- having already established the connection above via mutual TLS
using this same certificate -- might make the following HTTPS request
(with extra line breaks for display purposes only):

~~~
POST /token.oauth2 HTTP/1.1
Host: authz.example.net
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ax509-bearer
&assertion=MIIDXTCCAkWgAwIBAgIQEBAAAAAAAAAAAAAAAAAAADANBgkqhkiG9w0B
AQsFADBAMQswCQYDVQQGEwJVUzEUMBIGA1UEChMLRXhhbXBsZSBDb3Jw
[...omitted for brevity...]
~~~

# Interoperability Considerations {#interop}

Agreement between system entities regarding trust anchors, identifiers,
and endpoints is required in order to achieve interoperable deployments
of this profile. Specific items that require agreement are as follows:
the set of trust anchors and Issuing CAs recognized by the authorization
server, the rule used to map a certificate's Subject field or
subjectAltName extension to a resource owner identity or "client_id",
which Issuing CAs' certificates the authorization server considers itself
an intended relying party for (in lieu of an in-band "aud" equivalent, per
item 4 of {{format}}), the revocation-checking mechanism to be used, the
mechanism by which proof of possession is corroborated (direct mutual TLS
to the authorization server versus a trusted forwarding intermediary, per
item 7 of {{format}}), whether the authorization server accepts the
thumbprint reference form of {{thumbprint}} in addition to, or instead
of, a fully encoded certificate, and any maximum certificate lifetime the
authorization server will accept for use under this profile. The exchange
of such information is explicitly out of scope for this specification.
Where the Issuing CA is a workload identity control plane, some or all of
this agreement may be established and kept current automatically, for
example by having the authorization server obtain and refresh its set of
trust anchors from a SPIFFE trust bundle endpoint {{SPIFFE-X509-SVID}} or
an equivalent mechanism, rather than through static, manually configured
trust anchors; this specification does not define such a mechanism and
treats it as an implementation detail of how the out-of-band agreement
above is kept up to date.

Because certificates issued for general-purpose PKI use are typically
valid for periods far longer than is typical for a JWT or SAML bearer
assertion, deployments SHOULD either issue certificates intended for use
under this profile with comparatively short validity periods, or have the
authorization server apply a locally configured maximum acceptable
remaining lifetime, or both. Workload Identity Certificates
({{terminology}}) issued by systems such as SPIFFE/SPIRE
{{SPIFFE-X509-SVID}} or Athenz {{ATHENZ-X509}} already follow this
practice by design, typically being valid for periods on the order of
minutes to hours and rotated automatically well before expiry; deployments
using such certificates under this profile satisfy this recommendation
without additional configuration, and this is the expected deployment
pattern this specification is primarily designed around; see also
{{security}}.

RSA with SHA-256, and ECDSA with curve P-256 and SHA-256, are RECOMMENDED
baseline certificate signature algorithms for this profile, to maximize
interoperability.

# Security Considerations {#security}

The security considerations described within the following
specifications are all applicable to this document: "Assertion Framework
for OAuth 2.0 Client Authentication and Authorization Grants"
{{RFC7521}}, "The OAuth 2.0 Authorization Framework" {{RFC6749}}, and
"Internet X.509 Public Key Infrastructure Certificate and Certificate
Revocation List (CRL) Profile" {{RFC5280}}.

Unlike a JWT or SAML assertion, whose bearer semantics rely on the
issuer's signature over a short-lived, narrowly scoped set of claims, an
X.509 certificate is designed to be shared openly: it is routinely sent
in the clear during TLS handshakes, may be logged by intermediaries,
and, for publicly trusted CAs, may be published in Certificate
Transparency logs. A certificate is therefore not a secret, and a copy
of one proves nothing about who is presenting it. For this reason, this
specification requires, in item 7 of {{format}}, that possession of the
certificate's private key be corroborated for every request, either
through the TLS handshake to the authorization server itself or through
an attestation from a trusted intermediary. Implementations MUST NOT
grant access, and MUST NOT authenticate a client, solely on the basis
that a syntactically and cryptographically valid certificate was
presented as an "assertion" or "client_assertion" value; doing so would
allow trivial impersonation by any party that obtained a copy of the
certificate, without needing the associated private key.

Authorization servers SHOULD track recently processed
(Issuer, serialNumber) pairs, within a window derived from the
certificate's validity period or a locally configured limit, to reduce
the impact of a request being replayed even when proof of possession
corroboration per item 7 of {{format}} is somehow satisfied more than
once (for example, by a compromised or overly permissive forwarding
intermediary).

For general-purpose, longer-lived certificates, revocation checking is
particularly important under this profile, since certificate validity
periods can be much longer than the lifetime of a JWT or SAML bearer
assertion; a compromised private key can otherwise remain usable under
this profile for a much longer window than would be typical of the
analogous JWT or SAML bearer profiles. For Workload Identity Certificates
({{terminology}}), whose short validity periods and frequent, automatic
rotation are themselves the primary mitigation against a compromised
private key remaining usable for long, revocation checking is a useful
defense-in-depth measure but is less load-bearing than certificate
lifetime, and authorization servers deployed against such certificates
MAY choose not to perform revocation checking at all, provided the
configured maximum acceptable certificate lifetime (see {{interop}}) is
short enough that the risk is otherwise acceptable.

Authorization servers configured to trust more than one Issuing CA MUST
ensure that identifiers extracted from a certificate issued by one CA are
not confused with, or allowed to impersonate, identifiers meant to be
asserted only by a different CA.

The thumbprint reference form defined in {{thumbprint}} is only as
trustworthy as the source the authorization server checks it against. An
authorization server MUST compute the comparison thumbprint from a
certificate it obtained itself, either from its own TLS stack or from a
trusted intermediary via a channel the client cannot influence; it
MUST NOT accept a thumbprint together with a client-supplied certificate
purporting to match it, since a client able to influence both values
could reference an arbitrary certificate it does not hold the private
key for.

# Privacy Considerations

An X.509 certificate typically contains more identifying information
than the minimal set of claims a deployment would choose for a JWT or
SAML bearer assertion, and, being a long-lived, widely presented
credential, can act as a stable identifier that enables correlation of a
subject's activity across the multiple relying parties the certificate is
presented to. To prevent disclosure of such information to unintended
parties, an X.509 Bearer Certificate should only be transmitted over
encrypted channels, such as those provided by TLS {{RFC8446}}.

Deployments should determine the minimum amount of information necessary
in the Subject field and subjectAltName extension to complete the
exchange, should consider issuing purpose-specific certificates rather
than reusing a single general-purpose certificate across unrelated
relying parties, and should consult the guidance in {{RFC6973}} when
designing the identifiers a certificate exposes.

# IANA Considerations

## Sub-Namespace Registration of urn:ietf:params:oauth:grant-type:x509-bearer

This section registers the value "grant-type:x509-bearer" in the IANA
"OAuth URI" registry established by "An IETF URN Sub-Namespace for OAuth"
{{RFC6755}}.

* URN: urn:ietf:params:oauth:grant-type:x509-bearer
* Common Name: X.509 Bearer Certificate Grant Type Profile for OAuth 2.0
* Change Controller: IESG
* Specification Document: This document

## Sub-Namespace Registration of urn:ietf:params:oauth:client-assertion-type:x509-bearer

This section registers the value "client-assertion-type:x509-bearer" in
the IANA "OAuth URI" registry established by "An IETF URN Sub-Namespace
for OAuth" {{RFC6755}}.

* URN: urn:ietf:params:oauth:client-assertion-type:x509-bearer
* Common Name: X.509 Bearer Certificate Profile for OAuth 2.0 Client
  Authentication
* Change Controller: IESG
* Specification Document: This document

This document does not request assignment of an X.509 certificate
extension object identifier. Should a future revision define the
optional in-band audience-restriction extension mentioned in item 4 of
{{format}}, that revision will need to make an appropriate IANA request
at that time.

--- back

# Acknowledgments
{:numbered="false"}

This profile follows the pattern established by {{RFC7523}} and
{{RFC7522}}, which share a common lineage back to the OAuth Assertion
Framework {{RFC7521}}. TODO acknowledge reviewers and contributors.
