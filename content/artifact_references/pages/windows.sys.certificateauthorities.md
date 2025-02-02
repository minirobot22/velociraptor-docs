---
title: Windows.Sys.CertificateAuthorities
hidden: true
tags: [Client Artifact]
---

Certificate Authorities installed in Keychains/ca-bundles.

```yaml
name: Windows.Sys.CertificateAuthorities
description: Certificate Authorities installed in Keychains/ca-bundles.
sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'
    query: |
        SELECT Store, IsCA, Subject,
               encode(string=SubjectKeyId, type='hex') AS SubjectKeyId,
               encode(string=AuthorityKeyId, type='hex') AS AuthorityKeyId,
               Issuer, KeyUsageString,
               IsSelfSigned, SHA1, SignatureAlgorithm, PublicKeyAlgorithm, KeyStrength,
               NotBefore, NotAfter, HexSerialNumber
        FROM certificates()

```
