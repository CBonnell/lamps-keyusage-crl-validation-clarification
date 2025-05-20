---
title: "Clarification to processing Key Usage values during CRL validation"
abbrev: "CRL validation clarification"
category: std

docname: draft-ietf-lamps-keyusage-crl-validation-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
# area: AREA
# workgroup: WG Working Group
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
#  group: WG
#  type: Working Group
#  mail: WG@example.com
#  arch: https://example.com/WG
  github: "CBonnell/lamps-keyusage-crl-validation-clarification"
  latest: "https://CBonnell.github.io/ietf-lamps-keyusage-crl-validation-clarification/draft-ietf-lamps-keyusage-crl-validation.html"
updates: 5280

author:
 -
    fullname: "Corey Bonnell"
    organization: DigiCert, Inc.
    email: "corey.bonnell@digicert.com"
 -
    fullname:
      :: 伊藤 忠彦
      ascii: Tadahiko Ito
    organization: SECOM CO., LTD.
    email: tadahiko.ito.public@gmail.com
 -
    fullname:
      :: 大久保 智史
      ascii: Tomofumi Okubo
    organization: Penguin Securities Pte. Ltd.
    email: tomofumi.okubo+ietf@gmail.com

normative:

informative:


--- abstract

RFC 5280 defines the profile of X.509 certificates and certificate
revocation lists (CRLs) for use in the Internet. This profile requires
that certificates which certify keys for signing CRLs contain the key
usage extension with the `cRLSign` bit asserted. Additionally, RFC 5280
defines steps for the validation of CRLs. While there is a requirement
for CRL validators to verify that the `cRLSign` bit is asserted in the
`keyUsage` extension of the CRL issuer's certificate, this document
clarifies the requirement for relying parties to also verify the
presence of the `keyUsage` extension in the CRL issuer's certificate.
This check remediates a potential security issue that arises when
relying parties accept a CRL which is signed by a certificate with no
`keyUsage` extension, and therefore does not explicitly have the
`cRLSign` bit asserted.

--- middle

# Introduction

{{!RFC5280}} defines the profile of X.509 certificates and certificate
revocation lists (CRLs) for use in the Internet. Section 4.2.1.3 of
{{!RFC5280}} requires CRL issuer certificates to contain the `keyUsage`
extension with the `cRLSign` bit asserted. However, the CRL validation
algorithm specified in Section 6.3 of {{!RFC5280}} does not explicitly
include a corresponding check for the presence of the `keyUsage`
certificate extension. This document updates {{!RFC5280}} to require
that check.

{{the-issue}} describes the security concern that motivates this update.

{{crl-validation-algo-amendment}} updates the CRL validation algorithm
to resolve this concern.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# The risk of trusting CRLs signed with non-certified keys {#the-issue}

In some Public Key Infrastructures, entities are delegated by
Certification Authorities to sign CRLs. CRLs whose scope encompasses
certificates that have not been signed by the CRL issuer are known as
"indirect CRLs".

Certification Authorities delegate the issuance of CRLs
to other entities by issuing to the entity a certificate that asserts
the `cRLSign` bit in the `keyUsage` extension. The Certification
Authority will then sign certificates that fall within the scope of the
indirect CRL by including the `crlDistributionPoints` extension and
specifying the distinguished name ("DN") of the CRL issuer in the
`cRLIssuer` field of the corresponding distribution point.

The CRL issuer signs CRLs that assert the `indirectCRL` boolean within
the `issuingDistributionPoint` extension.

Applications which consume CRLs follow the validation algorithm as
specified in Section 6.3 of {{!RFC5280}}. In particular, Section 6.3.3
contains the following step for CRL validation:

> (f) Obtain and validate the certification path for the issuer of
    the complete CRL.  The trust anchor for the certification
    path MUST be the same as the trust anchor used to validate
    the target certificate.  If a `keyUsage` extension is present
    in the CRL issuer's certificate, verify that the `cRLSign` bit
    is set.

This step does not explicitly specify a check for the presence of the
`keyUsage` extension itself.

Additionally, the certificate profile in {{!RFC5280}} does not require
the inclusion of the `keyUsage` extension in a certificate if the
certified public key is not used for verifying the signatures of other
certificates or CRLs. Section 4.2.1.3 of {{!RFC5280}} says:

> Conforming CAs MUST include this extension in certificates that
   contain public keys that are used to validate digital signatures on
   other public key certificates or CRLs.

The allowance for the issuance of certificates without the `keyUsage`
extension and the lack of a check for the inclusion of the `keyUsage`
extension during CRL verification can manifest in a security issue. A
concrete example is described below.

1. The Certification Authority signs an end-entity CRL issuer
   certificate to subject `X` that certifies key `A` for signing CRLs by
   explicitly including the `keyUsage` extension and asserting the
   `cRLSign` bit in accordance with Section 4.2.1.3 of {{!RFC5280}}.
2. The Certification Authority signs one or more certificates that
   include the crlDistributionPoints extension with the DN for subject
   `X` included in the `cRLIssuer` field. This indicates that the
   CRL-based revocation information for these certificates will be
   provided by subject `X`.
3. The Certification Authority signs an end-entity certificate to
   subject `X` that certifies key `B`. This certificate contains no key
   usage extension, as the certified key is not intended to be used for
   signing CRLs and could be a “mundane” certificate of any type (e.g.,
   S/MIME, document signing certificate where the corresponding private
   key is stored on the filesystem of the secretary's laptop, etc.).
4. Subject `X` signs a CRL using key `B` and publishes the CRL at the
   `distributionPoint` specified in the `crlDistributionPoints`
   extension of the certificates signed in step 2.
5. Relying parties download the CRL published in step 4. The CRL
   validates successfully according to Section 6.3.3 of {{!RFC5280}},
   as the CRL issuer DN matches, and the check for the presence of the
   `cRLSign` bit in the `keyUsage` extension is skipped because the
   `keyUsage` extension is absent.

# Checking the presence of the `keyUsage` extension {#crl-validation-algo-amendment}

To remediate the security issue described in {{the-issue}}, this
document specifies the following amendment to step (f) of the CRL
algorithm as found in Section 6.3.3 of {{!RFC5280}}.

*OLD:*

> (f) Obtain and validate the certification path for the issuer of
    the complete CRL.  The trust anchor for the certification
    path MUST be the same as the trust anchor used to validate
    the target certificate.  If a `keyUsage` extension is present
    in the CRL issuer's certificate, verify that the `cRLSign` bit
    is set.

*NEW:*

> (f) Obtain and validate the certification path for the issuer of
    the complete CRL.  The trust anchor for the certification
    path MUST be the same as the trust anchor used to validate
    the target certificate.  Verify that the `keyUsage` extension is
    present in the CRL issuer's certificate and verify that the `cRLSign`
    bit is set.

# Security Considerations

If a Certification Authority has signed certificates to be used for
CRL verification but do not include the `keyUsage` extension in
accordance with Section 4.2.1.3 of {{!RFC5280}}, then relying party
applications that have implemented the modified verification algorithm
as specified in this document will be unable to verify CRLs signed by
the CRL issuer in question.

It is strongly RECOMMENDED that Certification Authorities include the
`keyUsage` extension in certificates to be used for CRL verification to
ensure that there are no interoperability issues where updated
applications are unable to verify CRLs.

If it is not possible to update the profile of CRL issuer certificates,
then the policy management authority of the affected Public Key
Infrastructure SHOULD update the subject naming requirements to ensure
that certificates to be used for different purposes contain unique DNs.

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
