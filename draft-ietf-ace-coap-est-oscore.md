---
title: Protecting EST Payloads with OSCORE
abbrev: EST-oscore
docname: draft-ietf-ace-coap-est-oscore-latest

ipr: trust200902
cat: std
submissiontype: IETF
area: Security
workgroup: ACE Working Group
keyword: Internet-Draft
coding: utf-8
pi: # can use array (if all yes) or hash here
  toc: yes
  sortrefs: yes
  symrefs: yes
  tocdepth: 2

author:
      -
        ins: G. Selander
        name: Göran Selander
        org: Ericsson AB
        email: goran.selander@ericsson.com
      -
        ins: S. Raza
        name: Shahid Raza
        org: University of Glasgow
        email: shahid.raza@glasgow.ac.uk
      -
        ins: M. Furuhed
        name: Martin Furuhed
        org: IN Groupe
        email: martin.furuhed@ingroupe.com
      -
        ins: M. Vučinić
        name: Mališa Vučinić
        org: Inria
        email: malisa.vucinic@inria.fr

normative:

  RFC2119:
  RFC6955:
  RFC8949:
  RFC7252:
  RFC7925:
  RFC7959:
  RFC9052:
  RFC9053:
  RFC8613:
  RFC9148:
  RFC9528:
  RFC8446:
  RFC9147:
  RFC9360:
  RFC9668:
  RFC8710:
  I-D.ietf-cose-cbor-encoded-cert:
  RFC2986:
  RFC5280:
  RFC8018:
  RFC5272:
  RFC9180:
  RFC5869:
  RFC8998:
  RFC5639:
  RFC9277:
  RFC6838:
  SP-800-227:
    target: https://doi.org/10.6028/NIST.SP.800-227
    title: Recommendation for Key-Encapsulation Mechanisms
    seriesinfo:
      "NIST": "Special Publication 800-227"
    author:
      -
        ins: G. Alagic
      -
        ins: E. Barker
      -
        ins: L. Chen
      -
        ins: D. Moody
      -
        ins: A. Roginsky
      -
        ins: H. Silberg
      -
        ins: N. Waller
    date: September 2025

informative:

  RFC6024:
  RFC7228:
  RFC7030:
  RFC9031:
  RFC9110:
  RFC9112:
  I-D.ietf-core-oscore-groupcomm:
  I-D.ietf-core-oscore-capable-proxies:

  SP-800-57:
    target: https://doi.org/10.6028/NIST.SP.800-57pt1r5
    title: Recommendation for Key Management
    seriesinfo:
      "NIST": "Special Publication 800-57 Revision 5"
    author:
      -
        ins: E. Barker
    date: May 2020

  SP-800-56A:
    target: https://doi.org/10.6028/NIST.SP.800-56Ar3
    title: Recommendation for Pair-Wise Key-Establishment Schemes Using Discrete Logarithm Cryptography
    seriesinfo:
      "NIST": "Special Publication 800-56A Revision 3"
    author:
      -
        ins: E. Barker
      -
        ins: L. Chen
      -
        ins: A. Roginsky
      -
        ins: A. Vassilev
      -
        ins: R. Davis
    date: April 2018


--- abstract

Enrollment over Secure Transport (EST) is a certificate provisioning protocol over HTTPS {{RFC7030}} or CoAPs {{RFC9148}}.
This document specifies how to carry EST over the Constrained Application Protocol (CoAP) protected with Object Security for Constrained RESTful Environments (OSCORE).
The specification builds on the EST-coaps {{RFC9148}} specification, but uses OSCORE and Ephemeral Diffie-Hellman over COSE (EDHOC) instead of DTLS.
The specification also leverages the certificate structures defined in {{I-D.ietf-cose-cbor-encoded-cert}}, which can be optionally used alongside X.509 certificates.

--- middle

# Introduction  {#intro}

One of the challenges with deploying a Public Key Infrastructure (PKI) for the Internet of Things (IoT) is certificate enrollment, because existing enrollment protocols are not optimized for constrained environments {{RFC7228}}.

One optimization of certificate enrollment targeting IoT deployments is specified in EST-coaps {{RFC9148}}, which defines a version of Enrollment over Secure Transport {{RFC7030}} for transporting EST payloads over CoAP {{RFC7252}} and DTLS {{RFC9147}}, instead of HTTP {{RFC9110}}{{RFC9112}} and TLS {{RFC8446}}.

This document describes a method for protecting EST payloads over CoAP with OSCORE {{RFC8613}}.
OSCORE specifies an extension to CoAP that protects messages at the application layer and can be applied independently of how CoAP messages are transported.
OSCORE can also be applied to CoAP-mappable HTTP, which enables end-to-end security for mixed CoAP and HTTP transfer of application layer data (see {{Section 11 of RFC8613}}).
Hence, EST payloads can be protected end-to-end independent of the underlying transport and through proxies translating between CoAP and HTTP.

OSCORE is designed for constrained environments, building on IoT standards such as CoAP, CBOR {{RFC8949}}, and COSE {{RFC9052}} {{RFC9053}}, and has in particular gained traction in settings where message sizes and the number of exchanged messages need to be kept at a minimum, such as 6TiSCH {{RFC9031}}, or for securing CoAP group messages {{I-D.ietf-core-oscore-groupcomm}}.
Where OSCORE is implemented and used for communication security, the reuse of OSCORE for other purposes, such as enrollment, reduces the code footprint.

Prior to running EST-oscore, the protocol defined in this specification, there must exist a trust relationship between the EST-oscore client and the EST-oscore server.
This trust relationship may be based on the pre-shared OSCORE security context, or on the common root of trust.
In case there is a pre-shared OSCORE security context, the CoAP exchange carrying EST payloads can occur immediately.
In case there is a common root of trust, a security handshake based on the Ephemeral Diffie-Hellman over COSE (EDHOC, {{RFC9528}}) protocol needs to occur prior to running EST-oscore.
How this trust relationship is established is out of the scope of this document.

How the EST-oscore server verifies the identity of the client prior to issuing a certificate is also out of the scope of this specification.

EST-oscore defines a number of optimizations with respect to EST-coaps:

* The DTLS record layer is replaced by OSCORE.
* The DTLS handshake is replaced by the lightweight authenticated key exchange protocol EDHOC {{RFC9528}}.
* Compact CBOR representations of X.509 certificates and EST payloads (see {{I-D.ietf-cose-cbor-encoded-cert}}) are optionally used.
* Certificates by reference (see {{RFC9360}}) are optionally used.
* The EST payloads protected by OSCORE can be proxied between constrained networks supporting CoAP and non-constrained networks supporting HTTP/HTTPs, through a CoAP-HTTP proxy without any security processing at the proxy (see {{proxying}}).
The concept "Registrar" and its required trust relationship with the EST server as described in {{Section 5 of RFC9148}} is therefore not applicable.

# Terminology   {#terminology}

{::boilerplate bcp14}

This document uses terminology from {{RFC9148}}, which in turn is based on {{RFC7030}} and, in turn, on {{RFC5272}}.

The term "Trust Anchor" follows the terminology of {{RFC6024}}:
"A trust anchor represents an authoritative entity via a public key and associated data.
The public key is used to verify digital signatures, and the associated data is used to constrain the types of information for which the trust anchor is authoritative."

Apart from enrolling certificates with keys that are used for signing, this document also specifies how to enroll certificates with keys that are used for Diffie-Hellman (DH) operations (static DH keys).
Instead of signing, possession of the private static DH key may be proved by generating a MAC given the recipient's public DH key.
Therefore, this document extends the definition of the term "Trust Anchor": the corresponding public key can also be used for MAC generation for static DH proof-of-possession procedures.

# Authentication

This specification replaces the DTLS handshake in EST-coaps with the lightweight authenticated key exchange protocol EDHOC {{RFC9528}}.
The enrollment using EST-oscore is based on the existence of an OSCORE Security Context protecting the messages conveying the EST payloads.
This Security Context is typically established through an EDHOC session preceding the initial enrollment.
Re-enrollment does not require a new EDHOC session.

The EST-oscore client MUST play the role of the EDHOC Initiator.
The EST-oscore server MUST play the role of the EDHOC Responder.

The EST-oscore clients and servers perform mutual authentication.
The EST server and EST client are responsible for ensuring that an acceptable cipher suite is negotiated.
The client authenticates the server before accepting any server response.
These requirements are fulfilled when using EDHOC {{RFC9528}}.

The server must also provide relevant information to the CA to support its decision about issuing a certificate.

## EDHOC

EDHOC supports authentication with certificates or raw public keys (referred to as "credentials"), and the credentials may either be transported in the protocol, or referenced.
This is determined by the identifier of the credential of the endpoint, ID_CRED_x for x= Initiator/Responder, which is transported in an EDHOC message.
This identifier may be the credential itself (in which case the credential is transported), or a pointer such as a URI of the credential (e.g., c5u, see {{I-D.ietf-cose-cbor-encoded-cert}}) or some other identifier which enables the receiving endpoint to retrieve the credential.

## Certificate-based Authentication

EST-oscore, like EST-coaps, supports certificate-based authentication between the EST client and server.
The client MUST be configured with an Implicit or Explicit Trust Anchor (TA) {{RFC7030}} database, enabling the client to authenticate the server.
The requirements on managing the Implicit and Explicit TA databases are discussed in {{Section 9.1 of RFC9148}} and {{Section 9.2 of RFC9148}} and apply to EST-oscore.

The EST client and EST server certificate SHOULD conform to {{RFC7925}}.
The EST client and/or EST server certificate MAY be a (natively signed) CBOR certificate {{I-D.ietf-cose-cbor-encoded-cert}}.
The EST client indicates its preference for the type of the certificate it supports through the CoAP Accept option included in the request to the EST server.

## Channel Binding {#channel-binding}

The {{RFC5272}} specification describes proof-of-possession as the ability of a client to prove its possession of a private key which is linked to a certified public key.
In case of a signature key, a proof-of-possession is generated by the client when it signs the PKCS#10 Request during the enrollment phase.
In case of a static DH key, a proof-of-possession is generated by the client when it generates a MAC and includes it in the PKCS#10 request, as per {{static-dh-keys}}.

Connection-based channel binding refers to the security binding between the PKCS#10 object and the underlying secure transport layer.
This is typically achieved by including the challengePassword attribute in the PKCS#10 object that is dependent on the underlying security session.
Connection-based proof-of-possession using the challengePassword attribute of the PKCS#10 object is OPTIONAL, see {{security-considerations}}.

## Optimizations {#optimizations}

This section contains optional behavior that may be used to reduce message sizes or round trips based on the application configuration.

* The third message of the EDHOC protocol, message_3, MAY be combined with an OSCORE-protected request {{RFC9668}}, enabling authenticated Diffie-Hellman key exchange and a protected CoAP request/response (which may contain an enrollment request and response) in two round trips {{RFC9668}}.

* The enrolled client certificate MAY be the CBOR-encoded certificates defined in {{I-D.ietf-cose-cbor-encoded-cert}}.

* The enrolled client certificate MAY be referenced instead of transported {{RFC9360}}.
The response to the PKCS#10 request MAY specify a reference to the enrolled certificate rather than the certificate itself (see {{certs-by-reference}}).

* The PKCS#10 object MAY request a certificate for a static DH key instead of a signature key.
This may result in a more compact request because the use of static DH keys may imply a proof-of-possession using a MAC, which is shorter than a signature.
Additionally, subsequent EDHOC sessions using static DH keys for authentication have less overhead than key exchange protocols using signature-based authentication credentials.

* When the EDHOC handshake precedes the enrollment request, it is RECOMMENDED for the EST-client to leverage the information from the EDHOC session on the selected cipher suite when making a decision on which type of credential to enroll.

# Protocol Design and Layering

EST-oscore uses CoAP {{RFC7252}} and Block-Wise transfer {{RFC7959}} to transfer EST messages in the same way as {{RFC9148}}.
Instead of the DTLS record layer, OSCORE {{RFC8613}} is used to protect the messages conveying the EST payloads.
External Authorization Data (EAD) fields of EDHOC messages are intentionally not used to carry EST payloads because EDHOC needs not be executed in the case of re-enrollment.
The DTLS handshake is replaced with EDHOC {{RFC9528}}.
{{fig-stack}} below shows the layered EST-oscore architecture.
Protocol design also allows that OSCORE and EDHOC messages are carried within the same CoAP message, as per {{RFC9668}}.

~~~~~~~~~~~ aasvg
             +----------------+
             |  EST messages  |
+------------+----------------+
|    EDHOC   |    OSCORE      |
+------------+----------------+
|        CoAP or HTTP         |
+-----------------------------+
~~~~~~~~~~~
{: #fig-stack title="The stack diagram of EST protected with OSCORE."}
{: artwork-align="center"}

EST-oscore follows much of the EST-coaps and EST design.
This includes the need to authenticate the EST-server before performing any request on the different EST endpoints specified in this document.

## Discovery and URI     {#discovery}

The discovery of EST resources and the definition of the short EST-coaps URI paths specified in {{Section 4.1 of RFC9148}}, as well as the new Resource Type defined in {{Section 8.2 of RFC9148}} apply to EST-oscore.
In a web link targeting a resource for EST-oscore, it is REQUIRED to indicate that the resource is only accessible using OSCORE, by means of the "osc" target attribute defined in {{Section 9 of RFC8613}}.

Example:

~~~~~~~~~~~

     REQ: GET /.well-known/core?rt=ace.est.sen

     RES: 2.05 Content
       Content-Format: 40 (application/link-format)
       Payload:
         </est>;rt=ace.est.sen;osc
~~~~~~~~~~~

## Mandatory/optional EST Functions {#est-functions}

The EST-oscore specification has the same set of required-to-implement functions as EST-coaps.
The content of {{table_functions}} is adapted from {{Section 4.2 of RFC9148}} and uses the updated URI paths (see {{discovery}}).
In addition, EST-oscore defines the /kemc function, which supports the enrollment of certificates for KEM keys and static DH keys (see {{kem-keys}}).

| EST functions  | EST-oscore implementation   |
|----------------|-----------------------------|
| /crts          | MUST                        |
| /sen           | MUST                        |
| /sren          | MUST                        |
| /skg           | OPTIONAL                    |
| /skc           | OPTIONAL                    |
| /att           | OPTIONAL                    |
| /kemc          | OPTIONAL                    |
{: #table_functions cols="l l" title="Mandatory and optional EST-oscore functions."}

### /crts {#crts}

EST-oscore provides the /crts operation.
A successful request from the client to this resource will be answered with one or more trust anchors, which are subsequently installed in the TA database (once the server is authenticated), resulting in Explicit TAs.

A trust anchor is commonly a self-signed certificate of the CA public key, of the format indicated by the CoAP Accept option present in the request.
In order to reduce transport overhead, the trust anchor could instead be a CBOR encoding of an X.509 certificate {{I-D.ietf-cose-cbor-encoded-cert}}.

The content of the response to a /crts request depends on the CoAP Accept option present in the request, as specified below.

If the /crts request does not include a CoAP Accept option, the server SHOULD respond with application/pkcs7-mime;smime-type=certs-only (see {{der}}).
Since a /crts request carries no payload, the server cannot infer a preference for CBOR-encoded objects from the encoding of the request, and the exception for CBOR-encoded requests in {{der}} does not apply.
An EST-oscore client that prefers a CBOR-encoded response MUST include a CoAP Accept option in the request.

When the EST-oscore server receives a /crts request with a CoAP Accept option indicating application/pkix-cert or application/cose-c509-cert+cbor, it MUST return only the single CA certificate of the CA that has issued, or will issue, the certificate of the current, authenticated EST-oscore client making the request.
An exception to this rule is when the server has been configured to operate exclusively with multiple CA trust anchors: then the server MUST respond with a 4.06 (Not Acceptable) error, to signal to the client that it needs to request another Content-Format that supports retrieval of multiple CA certificates, i.e., application/pkcs7-mime;smime-type=certs-only or application/cose-c509+cbor.

When the EST-oscore server receives a /crts request with a CoAP Accept option indicating application/pkcs7-mime;smime-type=certs-only or application/cose-c509+cbor without the "usage" parameter, it MUST return all the CA certificates that the client is expected to install as Explicit TAs.
The order of the certificates carries no meaning.

When the EST-oscore server receives a /crts request with a CoAP Accept option indicating application/cose-c509+cbor; usage=chain, it MUST return the chain of CA certificates of the CA that has issued, or will issue, the certificate of the current, authenticated EST-oscore client making the request.
The chain is ordered starting with the certificate of that CA and ending with the certificate of the root CA, i.e., the trust anchor.
If the CA that has issued, or will issue, the certificate of the client is itself the root CA, the chain consists of a single certificate.

## Payload formats

Similar to EST-coaps, EST-oscore allows transport of DER-encoded objects of a given Media-Type.
When transporting DER-encoded objects, EST-oscore uses the same CoAP Content-Format identifiers as EST-coaps when transferring EST requests and responses.
In addition, EST-oscore allows the transport of CBOR-encoded objects, as indicated by their corresponding Media-Type.

EST-oscore servers MUST support both the DER-encoded ASN.1 objects and the CBOR-encoded objects, i.e., they MUST support formats detailed in {{der}} and {{cbor}}.
It is up to the client to support only DER-encoded ASN.1, only CBOR encoding, or both.
Based on the client encoding of the CSR (DER encoding or CBOR encoding), the server is able to tell whether the client prefers a DER-encoded object ({{der}}) or a CBOR-encoded object ({{cbor}}) in response.
In addition, Content-Format negotiation for specific objects happens through the CoAP Accept option present in the requests.
The CoAP Accept option may not be present; this is a case which carries special semantics, see {{crts}}, {{der}}, and {{cbor}}.

### DER-encoded ASN.1 Objects {#der}

{{table_mediatype_asn1}} summarizes the information from {{Section 4.3 of RFC9148}} for what concerns the transport of DER-encoded ASN.1 objects.
The entries for /kemc are defined in this document (see {{kem-challenge}}).
The response to /kemc is a CBOR-encoded object, also for EST clients that use DER-encoded ASN.1 objects.

|  URI  | Media Type                                    | Type | #IANA |
|-------|-----------------------------------------------|------|-------|
| /crts | N/A                                           | req |   -   |
|       | application/pkix-cert                         | res |  287  |
|       | application/pkcs7-mime;smime-type=certs-only  | res |  281  |
| /sen  | application/pkcs10                            | req |  286  |
|       | application/pkix-cert                         | res |  287  |
|       | application/pkcs7-mime;smime-type=certs-only  | res |  281  |
| /sren | application/pkcs10                            | req |  286  |
|       | application/pkix-cert                         | res |  287  |
|       | application/pkcs7-mime;smime-type=certs-only  | res |  281  |
| /skg  | application/pkcs10                            | req |  286  |
|       | application/multipart-core                    | res |   62  |
| /skc  | application/pkcs10                            | req |  286  |
|       | application/multipart-core                    | res |   62  |
| /att  | N/A                                           | req |   -   |
|       | application/csrattrs                          | res |  285  |
| /kemc | application/pkix-spki                         | req | TBD-SPKI |
|       | application/kem-challenge+cbor                | res | TBD-KCH  |
{: #table_mediatype_asn1 cols="l l l l" title="EST functions and the associated ASN.1 CoAP Content-Format identifiers."}

Content-Format 281 and Content-Format 287 MUST be supported by EST-oscore servers.
It is up to the client to support only Content-Format 281, only Content-Format 287, or both.
As indicated in {{Section 4.3 of RFC9148}}, the client will use a CoAP Accept Option in the request to express the preferred response Content-Format.
If an Accept Option is not included in the request, the client is not expressing any preference and the server SHOULD choose format 281.
An exception to this "SHOULD" is in the case when the request contains a CBOR-encoded object (e.g., application/cose-c509-pkcs10+cbor), when the server SHOULD respond with a CBOR-encoded object (see {{cbor}}).

The generated response for /skg and /skc requests is an application/multipart-core object {{RFC8710}} with two parts: the private key and the corresponding certificate.
As specified in {{Section 4.8 of RFC9148}}, the parts of the application/multipart-core response do not have to be in a particular order, since each part is preceded by its Content-Format identifier; the order in {{table_cft_skg_skc}} is the one used in {{RFC9148}}.
{{Section 4.8 of RFC9148}} specifies that the private key in response to /skg or /skc requests may be either an encrypted (PKCS #7) or unencrypted (PKCS #8) key, depending on whether the CSR request included SMIMECapabilities.

Due to the use of OSCORE, which protects the communication between the EST client and the EST server end-to-end, it is possible to return the private key to /skc or /skg as an unencrypted PKCS #8 object (Content-Format identifier 284).
Therefore, when making the CSR to /skc or /skg, the EST client MUST NOT include SMIMECapabilities.
As a consequence, the private key part of the response to /skc or /skg is an unencrypted PKCS #8 object.

| Function | DER-encoded ASN.1 Response, Part 1 | DER-encoded ASN.1 Response, Part 2 |
|----------|------------------------------------|------------------------------------|
| /skg     | 284 | 281 |
| /skc     | 284 | 287 |
{: #table_cft_skg_skc cols="l l l" title="Response Content-Format identifiers for /skg and /skc in case of DER-encoded ASN.1 objects."}

### CBOR-encoded Objects {#cbor}

{{table_mediatype_cbor}} presents the equivalent information to {{der}} when CBOR-encoded objects are in use.

|  URI  | Media Type                                    | Type | #IANA |
|-------|-----------------------------------------------|------|-------|
| /crts | N/A                                           | req |   -   |
|       | application/cose-c509-cert+cbor               | res | TBD21 |
|       | application/cose-c509+cbor                    | res | TBD3  |
|       | application/cose-c509+cbor; usage=chain       | res | TBD15 |
| /sen  | application/cose-c509-pkcs10+cbor             | req | TBD4  |
|       | application/cose-c509-cert+cbor               | res | TBD21 |
|       | application/multipart-core                    | res |   62  |
| /sren | application/cose-c509-pkcs10+cbor             | req | TBD4  |
|       | application/cose-c509-cert+cbor               | res | TBD21 |
|       | application/multipart-core                    | res |   62  |
| /skg  | application/cose-c509-pkcs10+cbor             | req | TBD4  |
|       | application/cose-c509-pem+cbor                | res | TBD11 |
| /skc  | N/A                                           | req |   -   |
|       | N/A                                           | res |   -   |
| /att  | N/A                                           | req |   -   |
|       | application/cose-c509-crtemplate+cbor         | res | TBD19 |
| /kemc | application/cose-c509-pubkey+cbor             | req | TBD-C5PK |
|       | application/kem-challenge+cbor                | res | TBD-KCH  |
{: #table_mediatype_cbor cols="l l l l" title="EST functions and the associated CBOR CoAP Content-Format identifiers."}

Please note that {{Section 4.4 of I-D.ietf-cose-cbor-encoded-cert}} defines the format and the semantics of the response to /att.
The request to /kemc is defined in {{c509-pubkey}}, and the response is defined in {{kem-challenge}}.

Content-Format TBD21 MUST be supported by EST-oscore servers, and by EST-oscore clients that support CBOR-encoded objects.
A preference for any (future) Content-Format is to be expressed by the EST-client through the Accept option.

If a CoAP Accept option is not included in a request to /sen or /sren, the client is not expressing preference and the server SHOULD respond with a response application/multipart-core that includes the reference(s) to the enrolled certificate (e.g., c5t, c5u).
The application/multipart-core response MUST include the reference(s) to the enrolled certificate which allows the client or any other party to retrieve it (e.g., through a coap URI secured with OSCORE).
The application/multipart-core response MAY also include the actual certificate.
The exact contents of the application/multipart-core response are dependent on the application policy.
Because x5u and c5u references are of the same type application/cbor and the corresponding CoAP Content Format (60), mixing them in the same application/multipart-core response is not supported.
An exception to the "SHOULD" is in the case when the request contains a DER-encoded ASN.1 object (e.g., application/pkcs10), when the server SHOULD respond with an appropriate ASN.1 object (see {{der}}).

In the case of a request to /skg, the response is an application/cose-c509-pem+cbor object (Content-Format identifier TBD11), i.e., a C509PEM structure ({{Section 3.6 of I-D.ietf-cose-cbor-encoded-cert}}) containing the generated private key and the corresponding certificate.
The second element of the C509PEM structure MUST NOT be null and MUST contain the certificate enrolled for the generated private key.
If the second element contains more than one certificate, the first certificate MUST be the enrolled certificate.
The function /skc is not available when using CBOR-encoded objects, and clients MUST use the /skg function for server-side generated keys.

### C509 Public Key {#c509-pubkey}

This document defines the C509SubjectPublicKeyInfo structure, which corresponds to SubjectPublicKeyInfo in X.509 (see {{Section 4.1 of RFC5280}}).
It allows a C509 public key to be represented independently of a C509 certificate.

~~~~~~~~~~~ cddl
C509SubjectPublicKeyInfo = [
  subjectPublicKeyAlgorithm: AlgorithmIdentifier,
  subjectPublicKey:          Defined
]
~~~~~~~~~~~
{: #fig-C509SubjectPublicKeyInfoCDDL title="CDDL for C509SubjectPublicKeyInfo"}

A C509SubjectPublicKeyInfo is carried with the media type application/cose-c509-pubkey+cbor ({{cose-c509-pubkey}}) and the corresponding CoAP Content-Format TBD-C5PK ({{iana-content-formats}}).
A stored file format is defined with "magic number" TBD-C5PK-MAGIC using the reserved CBOR tag 55799 and the Content-Format TBD-C5PK, enveloped as described in {{Section 2.2 of RFC9277}}.

The fields subjectPublicKeyAlgorithm and subjectPublicKey have the same encoding and semantics as the corresponding fields of TBSCertificate in {{I-D.ietf-cose-cbor-encoded-cert}}.
In particular, the algorithm-specific encodings defined in {{I-D.ietf-cose-cbor-encoded-cert}} apply, e.g., the optimized encodings of RSA and elliptic curve public keys.

An elliptic curve public key is encoded as in a natively signed C509 certificate (type 2).
The leading bytes 0xfe and 0xfd MUST NOT be used.
When a Weierstrass-form elliptic curve public key is converted to an X.509 SubjectPublicKeyInfo, a point encoded with leading byte 0x02 or 0x03 is encoded as a compressed point, and a point encoded with leading byte 0x04 is encoded as an uncompressed point.

A C509SubjectPublicKeyInfo does not bind the public key to an identity.
An application that receives a public key in this form needs a separate means, such as a certificate or a pre-established trust relationship, to decide whether to trust the key.

## Message Bindings {#message-bindings}

The EST-oscore message characteristics are based on those specified in {{Section 4.4 of RFC9148}}.
The following applies:

  * EST-oscore endpoints MUST support delayed responses (see {{Section 4.7 of RFC9148}}).
  * EST-oscore endpoints MUST support the following CoAP options: OSCORE, Uri-Host, Uri-Path, Uri-Port, Content-Format, and Accept.
  * EST-oscore servers MUST implement Block1 and Block2.
  * EST-oscore clients MUST implement Block2 and MAY implement Block1.
  * The EST-coaps URLs based on coaps:// are translated to coap://, but with mandatory use of the CoAP OSCORE option.

## CoAP response codes

See {{Section 4.5 of RFC9148}}.

## Message Fragmentation

The EDHOC key exchange is optimized for low message overhead, in particular when using static DH keys instead of signature keys for authentication (e.g., method 3 of {{RFC9528}}).
Together with various measures listed in this document such as CBOR-encoded payloads {{RFC8949}}, CBOR certificates {{I-D.ietf-cose-cbor-encoded-cert}}, and certificates by reference ({{optimizations}}), a significant reduction of message sizes can be achieved.

Nevertheless, depending on the application, the protocol messages may become larger than the available frame size thus resulting in fragmentation and, in resource-constrained networks such as those based on IEEE 802.15.4 where throughput is limited, fragment loss can trigger costly retransmissions.

It is recommended to prevent 6LoWPAN fragmentation, since it involves an error-prone datagram reassembly.
To limit the size of the CoAP payload, this document specifies the requirements on implementing the CoAP options Block1 and Block2 (see {{message-bindings}}).

## Delayed Responses {#delayed-responses}

See {{Section 4.7 of RFC9148}}.

## Enrollment of Certificates with Static DH Keys {#static-dh-keys}

\[Note for authors: Remove the old text of this section, i.e., the proof-of-possession for static DH keys based on {{RFC6955}}, since the mechanism specified in {{kem-dhkem}} applies to all static DH keys.\]

This section specifies how the EST client enrolls a static DH key.
In general, a given key pair should only be used for a single purpose, such as key establishment, digital signature, or key transport.

The EST client attempting to enroll a DH key for a key usage operation other than digital signature can use an alternative proof-of-possession algorithm.
The EST client SHOULD prepare the PKCS#10 object and compute a MAC, replacing the signature, over the certification request information by following the steps in {{Section 6 of RFC6955}}.
The Key Derivation Function (KDF) and the MAC MUST be set to the HKDF and HMAC algorithms used by OSCORE.
The KDF and MAC is thus defined by the hash algorithm used by OSCORE in HKDF and HMAC, which by default is SHA-256.
When EDHOC is used, then the hash algorithm is the application hash algorithm of the selected cipher suite.

In some cases, it may be beneficial to exceptionally use the static DH private key associated with the public key used in enrollment for a one-time signing operation of the CSR.
While a key pair should only be used for a single purpose (e.g., key establishment or signing), this exceptional use for one-time signing of the CSR is allowed, as discussed in Section 5.6.3.2 of {{SP-800-56A}} and Section 5.2 of {{SP-800-57}}.

To generate a MAC according to the algorithm outlined in {{Section 6 of RFC6955}}, the client needs to know the public DH key of the proof-of-possession recipient/verifier, i.e., the EST server.
In the general case, the EST client MAY obtain the CA certs including the CA's DH certificate using the /crts function using an explicit request/response flow.
The obtained certificate indicates the DH group parameters which MUST be respected by the EST client when generating its own DH key pair.

As an optimization, when EDHOC precedes the enrollment and the optimized workflow based on the EDHOC + OSCORE combined request is being used as per {{Section 3 of RFC9668}}, the client MUST use the ephemeral public key of the EDHOC Responder, G_Y, as the recipient public key in the algorithm outlined in {{Section 6 of RFC6955}}.
When generating its DH key pair, the client uses the group parameters as indicated by the selected cipher suite used in the EDHOC session.

Instead of the mechanism specified in this section, the EST client SHOULD prove possession of the static DH private key through the challenge-response mechanism specified in {{kem-keys}}, in which the static DH public key is used as the public key of a DHKEM ({{kem-dhkem}}).
That mechanism requires neither a static DH key pair of the EST server nor the knowledge of such a key by the EST client.
The mechanism specified in this section remains applicable, e.g., to avoid the additional round trip to /kemc when the EST client can use the ephemeral public key G_Y of the EDHOC Responder as described above.

## Enrollment of Certificates with KEM Keys {#kem-keys}

This section specifies how the EST client enrolls a public key of a Key-Encapsulation Mechanism (KEM), such as ML-KEM.
A KEM key pair provides no signature operation, so the EST client cannot sign the certification request with the private key to be certified.
Instead, the EST client proves possession of the KEM private key through a challenge-response mechanism based on the unilateral key confirmation specified in Section 4.4 of {{SP-800-227}}.
The EST client (the decapsulator) is the key-confirmation provider, and the EST server (the encapsulator) is the key-confirmation recipient.
Together with the authentication of the EST client provided by OSCORE, this provides proof-of-possession as described in Section 4.5 of {{SP-800-227}}.
The mechanism also applies to static DH public keys, which are used as public keys of the corresponding DH-based KEM ({{kem-dhkem}}).

The mechanism applies to both PKCS#10 certification requests {{RFC2986}} and C509 certification requests ({{Section 4 of I-D.ietf-cose-cbor-encoded-cert}}).
It consists of the following steps:

1. The EST client sends its KEM public key to the /kemc resource of the EST server ({{kem-challenge}}).

2. The EST server encapsulates a shared secret key K to the KEM public key, and returns the resulting ciphertext in a KEM challenge ({{kem-challenge}}).

3. The EST client decapsulates K, computes a MAC over its certification request using K, and sends the certification request to /sen or /sren ({{kem-mac}}).
   Instead of the KEM public key, the certification request contains a reference to the KEM public key sent in step 1 ({{kem-keyref}}).

4. The EST server verifies the MAC using K, and issues the certificate for the KEM public key received in step 1 ({{kem-verification}}).

When a request to /sen or /sren uses the kemKeyRef algorithm ({{kem-keyref}}), the EST server MUST verify the MAC as specified in {{kem-verification}} instead of a signature.
Requests to /skg and /skc are not affected, since the EST server generates the key pair itself.

### KEM Challenge {#kem-challenge}

The EST client sends a POST request to /kemc, with a payload containing the KEM public key:

* An EST client that will send a C509 certification request sends a C509SubjectPublicKeyInfo ({{c509-pubkey}}) with Content-Format TBD-C5PK (application/cose-c509-pubkey+cbor).
  The public key is encoded as in the C509 certification request.

* An EST client that will send a PKCS#10 certification request sends a DER-encoded SubjectPublicKeyInfo {{RFC5280}} with Content-Format TBD-SPKI (application/pkix-spki).
  The public key is encoded as in the PKCS#10 certification request.

In the following, KemPublicKey denotes the public key as sent in the request, i.e., the C509SubjectPublicKeyInfo encoded with the core deterministic encoding requirements of {{Section 4.2.1 of RFC8949}}, or the DER-encoded SubjectPublicKeyInfo.

If the payload does not contain a KEM public key or a static DH public key, the EST server MUST respond with a 4.00 (Bad Request) error.
If the EST server does not support the algorithm of the public key, it MUST respond with a 5.01 (Not Implemented) error.

Otherwise, the EST server runs the encapsulation algorithm of the KEM with the public key, which produces a shared secret key K and a ciphertext c.
For a static DH public key, the KEM is the DHKEM specified in {{kem-dhkem}}.
Then, the EST server sends a 2.04 (Changed) response with Content-Format TBD-KCH (application/kem-challenge+cbor), whose payload is the KEM challenge:

~~~~~~~~~~~ cddl
KemChallenge = [
  keyId: bstr .size 32,
  serverId: bstr .size 32,
  encapsulation: bstr,
  lifetime: uint / null,
]
~~~~~~~~~~~

The fields are:

* keyId is the SHA-256 hash of the KemPublicKey received in the request.

* serverId is a 32-byte identifier of the EST server.
  When EDHOC is used, serverId is RECOMMENDED to be the SHA-256 hash of the authentication credential of the EST server, i.e., CRED_R or CRED_I ({{Section 3.5.2 of RFC9528}}).
  Otherwise, serverId is chosen by the EST server.

* encapsulation is the ciphertext c.

* lifetime is the number of seconds after which the EST server considers the KEM challenge expired, counted from the time the response is sent, or null if no lifetime is available.
  A relative value is used so that the EST client does not need a synchronized clock.
  If lifetime is null, the lifetime of the KEM challenge is determined by the policy of the EST server and is not communicated to the EST client.
  The EST client SHOULD send the certification request before the lifetime has elapsed; otherwise, it needs to request a new KEM challenge.

Until the KEM challenge is used or has expired, the EST server MUST retain the challenge state, which consists of at least the KemPublicKey, keyId, serverId, c, K, the identity of the authenticated EST client, and the expiry time of the KEM challenge.
A new KEM challenge replaces any outstanding KEM challenge for the same keyId and EST client.
To mitigate denial-of-service attacks, the EST server MUST enforce the expiry of KEM challenges and MUST limit the number of outstanding KEM challenges per EST client.

### PoP Nonce {#kem-nonce}

Since the key pair to be certified is a static key pair, the EST client, as the key-confirmation provider, contributes a nonce as its ephemeral data, as required by Section 4.4.1 of {{SP-800-227}}.
The nonce is carried in an attribute of the certification request, referred to as the PoP nonce attribute:

* In a PKCS#10 certification request, the PoP nonce attribute is the Sender Nonce attribute id-cmc-senderNonce ({{Section 6.6 of RFC5272}}), with a single value of type OCTET STRING of 32 bytes.
  It is used only as the contribution of the EST client to the key confirmation, and the EST server does not return a Recipient Nonce.

* In a C509 certification request, the PoP nonce attribute is the CR attribute "KEM PoP Nonce", with attributeType TBD-NONCE (see {{iana}}) and attributeValue of type C509KemPopNonce:

~~~~~~~~~~~ cddl
C509KemPopNonce = bstr .size 32
~~~~~~~~~~~

The EST client MUST generate a fresh random nonce for each certification request.
The length of 256 bits satisfies the requirement of Appendix C.2 of {{SP-800-227}} for all security strengths up to 256 bits.
The PoP nonce attribute MUST be present in a certification request that uses the kemKeyRef algorithm ({{kem-keyref}}).
The CR attribute "KEM PoP Nonce" MUST NOT be present in any other C509 certification request.

### Key Reference {#kem-keyref}

The EST client has already sent its KEM public key to the EST server in the request to /kemc.
To avoid sending it a second time, the certification request contains a reference to the KEM public key instead of the KEM public key itself.

To this end, this document defines the placeholder public key algorithm "kemKeyRef", which indicates that the subject public key is the KEM public key identified by keyId ({{kem-challenge}}):

* In a PKCS#10 certification request, the algorithm field of subjectPKInfo is the AlgorithmIdentifier id-alg-kemKeyRef with absent parameters, and the subjectPublicKey field of subjectPKInfo contains the keyId (32 bytes).

~~~~~~~~~~~ asn.1
id-alg-kemKeyRef OBJECT IDENTIFIER ::= {
  iso(1) identified-organization(3) dod(6) internet(1)
  security(5) mechanisms(5) pkix(7) algorithms(6) TBD-KEMREF-OID }
~~~~~~~~~~~

* In a C509 certification request, subjectPublicKeyAlgorithm is the integer TBD-KEMREF (see {{iana}}), and subjectPublicKey is the keyId:

~~~~~~~~~~~ cddl
KemKeyRefPublicKey = bstr .size 32
~~~~~~~~~~~

An EST-oscore client that uses the mechanism in this section MUST use the kemKeyRef algorithm in the certification request.
Since keyId is the SHA-256 hash of the KemPublicKey, and the MAC ({{kem-mac}}) covers the certification request information, the MAC binds the certification request to the KEM public key.

The kemKeyRef algorithm, i.e., id-alg-kemKeyRef or its C509 counterpart TBD-KEMREF, MUST NOT be used in certificates.
Before the certification request is passed to a CA or used for issuing the certificate, the EST server replaces the key reference with the KemPublicKey from the challenge state.
Since the EST server verifies the proof-of-possession ({{kem-verification}}), it acts as a Registration Authority for the CA in this respect.

### MAC Computation by the EST Client {#kem-mac}

The EST client decapsulates the ciphertext in the encapsulation field of the KEM challenge with its private key, and obtains the shared secret key K.
The key-confirmation key KC_Key is K in its entirety (Section 4.4.2 of {{SP-800-227}}), i.e., no data key is derived.

The EST client constructs the certification request information, i.e., the CertificationRequestInfo of a PKCS#10 certification request or the TBSCertificationRequest of a C509 certification request, including the key reference ({{kem-keyref}}) and the PoP nonce attribute.
Then, the EST client constructs MAC_Data as specified in Section 4.4.1 of {{SP-800-227}}:

~~~~~~~~~~~
MAC_Data = KC_Step_Label || ID_P || ID_R || Eph_P || Eph_R
           || Extra_P || Extra_R
~~~~~~~~~~~

where:

* KC_Step_Label is the 6-byte ASCII string "KC_1_D", indicating that the decapsulator provides the first key-confirmation message;

* ID_P is keyId (32 bytes), identifying the KEM key pair of the EST client;

* ID_R is serverId (32 bytes) from the KEM challenge;

* Eph_P is the value of the PoP nonce attribute (32 bytes);

* Eph_R is the ciphertext c from the encapsulation field of the KEM challenge, whose length is fixed by the KEM;

* Extra_P is the DER-encoded CertificationRequestInfo of a PKCS#10 certification request, or the CBOR-encoded TBSCertificationRequest of a C509 certification request; and

* Extra_R is empty.

All fields except Extra_P have a fixed length, so MAC_Data can be parsed unambiguously.

The EST client computes the MAC tag over MAC_Data with KC_Key, and places it in the certification request as follows:

* In a PKCS#10 certification request, the signatureAlgorithm field is the AlgorithmIdentifier of the MAC algorithm (e.g., id-hmacWithSHA256 {{RFC8018}} for HMAC-SHA256), and the signature field contains the MAC tag.

* In a C509 certification request, subjectSignatureAlgorithm identifies the MAC algorithm (e.g., TBD-HMAC for HMAC-SHA256, see {{iana}}), and subjectSignatureValue contains the MAC tag.

The EST client then destroys K and sends the certification request to /sen or /sren.

The MAC algorithm MUST be one of the approved MAC algorithms for key confirmation listed in Appendix C.1 of {{SP-800-227}}, and the length of KC_Key MUST be permitted for that algorithm.
EST servers MUST support HMAC-SHA256.
EST clients SHOULD use HMAC-SHA256, and MAY use another approved MAC algorithm, e.g., if required by the security strength of the KEM.

### Verification by the EST Server {#kem-verification}

When receiving a request to /sen or /sren that uses the kemKeyRef algorithm, the EST server:

1. takes keyId from the subjectPublicKey field of the certification request, and verifies that it is 32 bytes long;

2. retrieves the challenge state for keyId, and verifies that:

   * the KEM challenge has not expired;

   * the request is sent by the authenticated EST client to which the KEM challenge was issued; and

   * the KemPublicKey has the format that corresponds to the certification request, i.e., a C509SubjectPublicKeyInfo for a C509 certification request, or a DER-encoded SubjectPublicKeyInfo for a PKCS#10 certification request;

3. verifies that the PoP nonce attribute is present and that its value is 32 bytes long;

4. verifies that it supports the MAC algorithm indicated in the certification request;

5. constructs MAC_Data as specified in {{kem-mac}}, using serverId and c from the challenge state; and

6. verifies the MAC tag over MAC_Data, using the indicated MAC algorithm and K from the challenge state.

Each KEM challenge MUST be used at most once, i.e., the EST server MUST delete the challenge state, including K, after the first request to /sen or /sren that refers to it, regardless of whether the verification succeeds.
If any step fails, the EST server MUST reject the request with a 4.00 (Bad Request) error.
Otherwise, the EST server replaces the key reference with the KemPublicKey from the challenge state ({{kem-keyref}}), and proceeds with issuing the certificate as for a signed certification request.

### Static DH Keys {#kem-dhkem}

A static DH public key is used as the public key of a DH-based KEM (DHKEM), as specified in {{Section 4.1 of RFC9180}}.
{{table_dhkem}} lists the DHKEM for each type of static DH public key, together with its KEM identifier (kem_id), the length Nenc of the ciphertext, and the length Nsecret of the shared secret (see {{Section 4 of RFC9180}}).

| Static DH public key                 | DHKEM                               | kem_id        | Nenc | Nsecret |
|--------------------------------------|-------------------------------------|---------------|------|---------|
| EC public key with secp256r1 (P-256) | DHKEM(P-256, HKDF-SHA256)           | 0x0010        | 65   | 32      |
| EC public key with secp384r1 (P-384) | DHKEM(P-384, HKDF-SHA384)           | 0x0011        | 97   | 48      |
| EC public key with secp521r1 (P-521) | DHKEM(P-521, HKDF-SHA512)           | 0x0012        | 133  | 64      |
| EC public key with sm2p256v1 (SM2)   | DHKEM(SM2, HKDF-SM3)                | TBD-SM2-KEM   | 65   | 32      |
| EC public key with brainpoolP256r1   | DHKEM(brainpoolP256r1, HKDF-SHA256) | TBD-BP256-KEM | 65   | 32      |
| EC public key with brainpoolP384r1   | DHKEM(brainpoolP384r1, HKDF-SHA384) | TBD-BP384-KEM | 97   | 48      |
| EC public key with brainpoolP512r1   | DHKEM(brainpoolP512r1, HKDF-SHA512) | TBD-BP512-KEM | 129  | 64      |
| X25519                               | DHKEM(X25519, HKDF-SHA256)          | 0x0020        | 32   | 32      |
| X448                                 | DHKEM(X448, HKDF-SHA512)            | 0x0021        | 56   | 64      |
{: #table_dhkem cols="l l l r r" title="DHKEM for static DH public keys."}

The DHKEMs for SM2 and for the Brainpool curves are not specified in {{RFC9180}}, and are defined in this document.
Each of them is specified like DHKEM(P-256, HKDF-SHA256) in {{Sections 4.1 and 7.1 of RFC9180}}, with the following differences:

* The elliptic curve is sm2p256v1 {{RFC8998}}, brainpoolP256r1, brainpoolP384r1, or brainpoolP512r1 {{RFC5639}}, respectively.
  SerializePublicKey, DeserializePublicKey, SerializePrivateKey, DeserializePrivateKey, and the DH function are as specified for P-256, i.e., public keys are encoded in the uncompressed format, private keys are encoded as fixed-length big-endian integers of length Nsk, and the result of DH is the x-coordinate of the shared point, encoded as a fixed-length big-endian integer of length Nsk.

* In DeriveKeyPair, the order is the order of the respective curve, and the bitmask is 0xff.

* The KDF is HKDF {{RFC5869}} with the hash function SM3 {{RFC8998}} for DHKEM(SM2, HKDF-SM3), and with SHA-256, SHA-384, or SHA-512 for DHKEM(brainpoolP256r1, HKDF-SHA256), DHKEM(brainpoolP384r1, HKDF-SHA384), or DHKEM(brainpoolP512r1, HKDF-SHA512), respectively.

* The values of Nsecret, Nenc, Npk, and Nsk are given in {{table_new_dhkem}}.

| KEM                                 | kem_id        | Nsecret | Nenc | Npk | Nsk |
|-------------------------------------|---------------|---------|------|-----|-----|
| DHKEM(SM2, HKDF-SM3)                | TBD-SM2-KEM   | 32      | 65   | 65  | 32  |
| DHKEM(brainpoolP256r1, HKDF-SHA256) | TBD-BP256-KEM | 32      | 65   | 65  | 32  |
| DHKEM(brainpoolP384r1, HKDF-SHA384) | TBD-BP384-KEM | 48      | 97   | 97  | 48  |
| DHKEM(brainpoolP512r1, HKDF-SHA512) | TBD-BP512-KEM | 64      | 129  | 129 | 64  |
{: #table_new_dhkem cols="l l r r r r" title="Parameters of the DHKEMs defined in this document."}

The EST server runs Encap(pkR) of the DHKEM, where pkR is the static DH public key of the EST client, and obtains the shared secret K (shared_secret) and the ciphertext c (enc).
The EST client runs Decap(enc, skR) of the DHKEM, where skR is its static DH private key, and obtains K.
Within Encap and Decap, public keys are serialized with SerializePublicKey of the DHKEM ({{Section 7.1.1 of RFC9180}}).
This does not affect the encoding of the public key in the request to /kemc, in keyId, or in the certificate.
The lengths of c and K are Nenc and Nsecret, respectively, as given in {{table_dhkem}}.
All other steps are as specified for KEM public keys.

EST-oscore clients SHOULD use the mechanism in this section to enroll static DH keys.
Unlike the proof-of-possession specified in {{static-dh-keys}}, it does not require the EST client to know a DH public key of the EST server, e.g., from a DH certificate of the EST server or the ephemeral public key G_Y of the EDHOC Responder.
In particular, it also works when the curve of the key to be certified differs from the curve used in EDHOC.
On the other hand, it requires an additional round trip to /kemc.

## Enrollment of Certificates by Reference {#certs-by-reference}

The EST client may indicate preference for enrolling a certificate by reference.
There are two cases to distinguish: 1) any certificate reference, or 2) a specific Content-Format.
In the first case, the EST client indicates preference for receiving any certificate by reference by sending a CBOR-encoded request without the CoAP Accept option.
In the second case, the EST client includes a Content-Format identifier in the CoAP Accept option indicating preference for receiving a specific reference (e.g., application/cose-certhash+cbor, application/cose-certhash+cbor;usage=c509, application/cbor containing a URI {{I-D.ietf-cose-cbor-encoded-cert}}).
It is out of the scope of this specification how the certificate by reference gets resolved to the actual certificate by other parties participating in the communication with the EST client.

# HTTP-CoAP Proxy {#proxying}

As noted in {{Section 5 of RFC9148}}, in real-world deployments, the EST server will not always reside within the CoAP boundary.
The EST-server can be outside the constrained network in a non-constrained network that supports HTTP but not CoAP, thus requiring an intermediary CoAP-to-HTTP proxy.

Since OSCORE is applicable to CoAP-mappable HTTP (see {{Section 11 of RFC8613}}) the messages conveying the EST payloads can be protected end-to-end between the EST client and EST server, irrespective of the transport protocol or potential transport layer security that may need to be terminated at the proxy, see {{fig-proxy}}.
Therefore, the concept "Registrar" and its required trust relationship with EST server as described in {{Section 5 of RFC9148}} is not applicable.

The mappings between CoAP and HTTP referred to in {{Section 8.1 of RFC9148}} apply, and additional mappings resulting from the use of OSCORE are specified in {{Section 11 of RFC8613}}.

OSCORE provides end-to-end security between the EST Server and EST Client.
If a secure association is needed between the EST Client and the CoAP-to-HTTP Proxy, this may also rely on OSCORE {{I-D.ietf-core-oscore-capable-proxies}}.

~~~~~~~~~~~ aasvg
                                       Constrained-Node Network
  .---------.                      .-----------------------------.
  |   CA    |                      |                             |
  '---------'                      |                             |
       |                           |                             |
   .------.  HTTP   .-----------------.   CoAP   .-----------.   |
   | EST  |<------->|  CoAP-to-HTTP   |<-------->| EST Client|   |
   |Server|  (TLS)  |      Proxy      |          '-----------'   |
   '------'         '-----------------'                          |
                                   |                             |
       <------------------------------------------------>        |
                        OSCORE     |                             |
                                   |                             |
                                   '-----------------------------'
~~~~~~~~~~~
{: #fig-proxy title="CoAP-to-HTTP proxy at the CoAP boundary."}
{: artwork-align="center"}


# Security Considerations  {#security-considerations}

## Server-generated Private Keys

This document enables the EST client to request the generation of private keys and the enrollment of the corresponding public key through /skg and /skc functions.
As discussed in {{Section 9 of RFC9148}}, the transport of private keys generated at the EST-server is inherently risky.
The use of server-generated private keys may lead to the increased probability of digital identity theft.
Therefore, implementations SHOULD NOT use server-generated private key EST functions.

A cryptographically-secure pseudo-random number generator is required to be available to generate good quality private keys on EST-clients.
A cryptographically-secure pseudo-random number generator is also a dependency of many security protocols.
This includes the EDHOC protocol, which EST-oscore uses for the mutual authentication of EST-client and EST-server.
If EDHOC is used and a secure pseudo-random number generator is available, the EST-client MUST NOT use server-generated private key EST functions.
However, EST-oscore also allows pre-shared OSCORE security contexts to be used for authentication, meaning that EDHOC may not necessarily be present in the protocol stack of an EST-client.
If EDHOC is not used for authentication, and the EST-client device does not have a cryptographically secure pseudo-random number generator, then the EST-client MAY use the server-generated private key functions.

Although hardware random number generators are becoming dominantly present in modern IoT devices, it has been shown that many available hardware modules contain vulnerabilities and do not produce cryptographically secure random numbers.
It is therefore important to use multiple randomness sources to seed the cryptographically secure pseudo-random number generator.

## Considerations on Channel Binding

{{Section 3 of RFC9148}} specifies that the use of connection-based channel binding is optional, and achieves it by including the tls-unique value in the CSR.
As a special case, when used with EDHOC for the enrollment of static DH keys, this specification achieves connection-based channel binding by using the EDHOC ephemeral public key of the Responder as the public key in the proof-of-possession algorithm that generates a PKCS#10 MAC.
Therefore, connection-based channel binding is in this case achieved without any additional overhead.

Other cases include pre-shared OSCORE security contexts and the case where the signature key used for signing the CSR is different from the key used in the EDHOC session.
In these other cases, this specification makes explicit channel binding based on the challengePassword attribute in PKCS#10 requests OPTIONAL.
For example, the challengePassword attribute could be used for freshness in the case of pre-shared OSCORE security contexts and of a re-enrollment request.

EST-servers MUST support the challengePassword attribute in PKCS#10 requests.
How challengePassword is processed is outside of the scope of this specification and can be specified by an application policy.

# IANA Considerations  {#iana}

## SMI Security for PKIX Algorithms Registry

IANA is asked to allocate the following object identifier in the "SMI Security for PKIX Algorithms" registry (1.3.6.1.5.5.7.6):

| Decimal        | Description      | References |
|----------------|------------------|------------|
| TBD-KEMREF-OID | id-alg-kemKeyRef | RFC XXXX   |
{: #table_iana_kemref_oid cols="l l l" title="Object identifier for the kemKeyRef algorithm."}

## C509 Public Key Algorithms Registry

IANA is asked to register the following entry in the "C509 Public Key Algorithms" registry under the registry group "CBOR Encoded X.509 (C509)", defined in {{I-D.ietf-cose-cbor-encoded-cert}}.

~~~~~~~~~~~
+------------+--------------------------------------------------------+
| Value      | Public Key Algorithm                                   |
+============+========================================================+
| TBD-KEMREF | Name:        KEM Key Reference                         |
|            | Identifiers: id-alg-kemKeyRef                          |
|            | OID:         1.3.6.1.5.5.7.6.TBD-KEMREF-OID            |
|            | Parameters:  Absent                                    |
|            | DER:         TBD                                       |
|            | Comments:    Only in certification requests for KEM    |
|            |              public keys. subjectPublicKey is the      |
|            |              keyId (bstr .size 32). MUST NOT be used   |
|            |              in certificates.                          |
|            | Reference:   [RFC-XXXX]                                |
+------------+--------------------------------------------------------+
~~~~~~~~~~~

## HPKE KEM Identifiers Registry

IANA is asked to register the following entries in the "HPKE KEM Identifiers" registry within the "Hybrid Public Key Encryption (HPKE)" registry group, for the DHKEMs defined in {{kem-dhkem}}.

| Value         | KEM                                 | Nsecret | Nenc | Npk | Nsk | Auth | Reference |
|---------------|-------------------------------------|---------|------|-----|-----|------|-----------|
| TBD-SM2-KEM   | DHKEM(SM2, HKDF-SM3)                | 32      | 65   | 65  | 32  | yes  | RFC XXXX  |
| TBD-BP256-KEM | DHKEM(brainpoolP256r1, HKDF-SHA256) | 32      | 65   | 65  | 32  | yes  | RFC XXXX  |
| TBD-BP384-KEM | DHKEM(brainpoolP384r1, HKDF-SHA384) | 48      | 97   | 97  | 48  | yes  | RFC XXXX  |
| TBD-BP512-KEM | DHKEM(brainpoolP512r1, HKDF-SHA512) | 64      | 129  | 129 | 64  | yes  | RFC XXXX  |
{: #table_iana_hpke_kem cols="l l r r r r l l" title="HPKE KEM identifiers for the DHKEMs defined in this document."}

## C509 Signature Algorithms Registry

IANA is asked to register the following entry in the "C509 Signature Algorithms" registry under the registry group "CBOR Encoded X.509 (C509)", defined in {{I-D.ietf-cose-cbor-encoded-cert}}.

~~~~~~~~~~~
+----------+----------------------------------------------------------+
| Value    | Signature Algorithm                                      |
+==========+==========================================================+
| TBD-HMAC | Name:        HMAC-SHA256                                 |
|          | Identifiers: id-hmacWithSHA256                           |
|          | OID:         1.2.840.113549.2.9                          |
|          | Parameters:  NULL                                        |
|          | DER:         30 0C 06 08 2A 86 48 86 F7 0D 02 09 05 00   |
|          | Comments:    Only for the KEM proof-of-possession in     |
|          |              certification requests. MUST NOT be used in |
|          |              certificates.                               |
|          | Reference:   [RFC-XXXX]                                  |
+----------+----------------------------------------------------------+
~~~~~~~~~~~

## C509 CR Attributes Registry

IANA is asked to register the following entry in the "C509 CR Attributes" registry under the registry group "CBOR Encoded X.509 (C509)", defined in {{I-D.ietf-cose-cbor-encoded-cert}}.

~~~~~~~~~~~
+-----------+---------------------------------------------------------+
| Value     | CR Attribute                                            |
+===========+=========================================================+
| TBD-NONCE | Name:            KEM PoP Nonce                          |
|           | Identifiers:     N/A                                    |
|           | OID:             N/A                                    |
|           | DER:             N/A                                    |
|           | Comments:        Nonce for the KEM proof-of-possession  |
|           |                  in C509 certification requests         |
|           | attributeValue:  C509KemPopNonce                        |
|           | Reference:       [RFC-XXXX]                             |
+-----------+---------------------------------------------------------+
~~~~~~~~~~~

## Media Types Registry

IANA is asked to register the following media types in the "application" registry in the "Media Types" registry group, following the procedures in {{RFC6838}}.

### application/cose-c509-pubkey+cbor {#cose-c509-pubkey}

When the application/cose-c509-pubkey+cbor media type is used, the data is a C509SubjectPublicKeyInfo structure ({{c509-pubkey}}).

* Type name: application
* Subtype name: cose-c509-pubkey+cbor
* Required parameters: N/A
* Optional parameters: N/A
* Encoding considerations: binary (CBOR)
* Security considerations: See {{security-considerations}} of RFC XXXX.
* Interoperability considerations: N/A
* Published specification: RFC XXXX
* Applications that use this media type: Applications that employ COSE and use C509 public keys.
* Fragment identifier considerations: N/A
* Additional information:
  * Deprecated alias names for this type: N/A
  * Magic number(s): TBD-C5PK-MAGIC
  * File extension(s): .c509
  * Macintosh file type code(s): N/A
* Person & email address to contact for further information: iesg@ietf.org
* Intended usage: COMMON
* Restrictions on usage: N/A
* Author/Change controller: IETF

### application/pkix-spki

When the application/pkix-spki media type is used, the data is a DER-encoded SubjectPublicKeyInfo {{RFC5280}}.

* Type name: application
* Subtype name: pkix-spki
* Required parameters: N/A
* Optional parameters: N/A
* Encoding considerations: binary (DER)
* Security considerations: See {{security-considerations}} of RFC XXXX.
* Interoperability considerations: N/A
* Published specification: RFC XXXX
* Applications that use this media type: EST clients requesting a KEM challenge, see {{kem-challenge}} of RFC XXXX.
* Fragment identifier considerations: N/A
* Additional information:
  * Deprecated alias names for this type: N/A
  * Magic number(s): N/A
  * File extension(s): N/A
  * Macintosh file type code(s): N/A
* Person & email address to contact for further information: iesg@ietf.org
* Intended usage: COMMON
* Restrictions on usage: N/A
* Author/Change controller: IETF

### application/kem-challenge+cbor

When the application/kem-challenge+cbor media type is used, the data is a KemChallenge structure ({{kem-challenge}}).

* Type name: application
* Subtype name: kem-challenge+cbor
* Required parameters: N/A
* Optional parameters: N/A
* Encoding considerations: binary (CBOR)
* Security considerations: See {{security-considerations}} of RFC XXXX.
* Interoperability considerations: N/A
* Published specification: RFC XXXX
* Applications that use this media type: EST servers issuing a KEM challenge, see {{kem-challenge}} of RFC XXXX.
* Fragment identifier considerations: N/A
* Additional information:
  * Deprecated alias names for this type: N/A
  * Magic number(s): N/A
  * File extension(s): N/A
  * Macintosh file type code(s): N/A
* Person & email address to contact for further information: iesg@ietf.org
* Intended usage: COMMON
* Restrictions on usage: N/A
* Author/Change controller: IETF

## CoAP Content-Formats Registry {#iana-content-formats}

IANA is asked to register the following entries in the "CoAP Content-Formats" registry within the "Constrained RESTful Environments (CoRE) Parameters" registry group.

| Content Type                      | Content Coding | ID       | Reference |
|-----------------------------------|----------------|----------|-----------|
| application/cose-c509-pubkey+cbor | -              | TBD-C5PK | RFC XXXX  |
| application/pkix-spki             | -              | TBD-SPKI | RFC XXXX  |
| application/kem-challenge+cbor    | -              | TBD-KCH  | RFC XXXX  |
{: #table_iana_cf cols="l l l l" title="CoAP Content-Format identifiers."}

## Resource Type (rt=) Link Target Attribute Values Registry

IANA is asked to register the following entry in the "Resource Type (rt=) Link Target Attribute Values" registry within the "Constrained RESTful Environments (CoRE) Parameters" registry group.

| Value        | Description            | Reference |
|--------------|------------------------|-----------|
| ace.est.kemc | KEM challenge issuance | RFC XXXX  |
{: #table_iana_rt cols="l l l" title="Resource Type for the /kemc function."}

RFC Editor: Please replace "RFC XXXX" and "RFC-XXXX" with the RFC number of this specification and delete this paragraph.

--- back

# Example Enrollment With Optimizations

The message flow starts with the EST client sending EDHOC message_1.
The EDHOC handshake follows and concludes with the EDHOC message_3.
EDHOC message_3 is carried in the same message as the OSCORE enrollment request, as specified in {{RFC9668}}.
The OSCORE enrollment request contains a CoAP POST to the /sen endpoint.
This POST request includes the Content-Format option set to the value application/cose-c509-pkcs10+cbor, and the Accept option set to the value application/cose-c509-cert+cbor, indicating the support for CBOR-encoded objects.
In response, the client receives the application/cose-c509-cert+cbor object that contains the certificate.

~~~~~~~~~~~ aasvg

EST Client                   EST Server
|                                     |
|     EDHOC message_1                 |
+------------------------------------>|
|     EDHOC message_2                 |
|<------------------------------------+
|     EDHOC message_3                 |
|            +                        |
|     OSCORE request to /sen          |
+------------------------------------>|
|       POST /.well-known/est/sen     |
|       Accept: TBD21                 |
|       Content-Format: TBD4          |
|       Payload: cose-c509-pkcs10+cbor|
|                                     |
|     OSCORE response                 |
|<------------------------------------+
|       2.04 Changed                  |
|       Content-Format: TBD21         |
|       Payload: cose-c509-cert+cbor  |

~~~~~~~~~~~
{: #fig-sen-optimizations title="Enrollment EST-oscore flow with optimizations."}
{: artwork-align="center"}

# Acknowledgments
{: numbered="no"}

The authors would like to thank Timothy Claeys for his work on the initial versions of this document.

The authors would also like to thank
{{{Esko Dijk}}},
{{{Marco Tiloca}}}, and
{{{John Mattsson}}}
for providing a review of the document.

Work on this document has in part been supported by the Horizon Europe Framework Programme project OpenSwarm (grant agreement No. 101093046).

--- fluff


