# ADR: Email Platform with Stalwart

**Status:** Accepted
**Date:** 2024-07-01

## Context

Need email capability for transactional emails and notifications.

## Decision

Use Stalwart as self-hosted email server.

## Rationale

| Option | Cost | Protocols | Self-Hosted |
|--------|------|-----------|-------------|
| Cloud email (SES) | $$/volume | SMTP only | ❌ |
| Postfix/Dovecot | $0 | SMTP/IMAP | Complex |
| **Stalwart** | $0 | JMAP/IMAP/SMTP | Simple | **Selected** |

**Key Decision Factors:**
- Modern JMAP protocol support
- All-in-one (MTA + MDA + webmail)
- Rust-based (low resources)
- Easy Kubernetes deployment

## Features

| Feature | Support |
|---------|---------|
| SMTP (sending) | ✅ |
| IMAP (mailboxes) | ✅ |
| JMAP (modern API) | ✅ |
| Spam filtering | ✅ SpamAssassin compatible |
| DKIM signing | ✅ |
| Rate limiting | ✅ |

## Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stalwart
  namespace: workplace
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: stalwart
          image: stalwartlabs/mail-server:latest
          ports:
            - containerPort: 25    # SMTP
            - containerPort: 587   # Submission
            - containerPort: 993   # IMAPS
            - containerPort: 8080  # JMAP/Webmail
```

## DNS Requirements

| Record | Type | Value |
|--------|------|-------|
| `mail.<domain>` | A | <node-ip> |
| `<domain>` | MX | mail.<domain> |
| `<domain>` | TXT | SPF record |
| `default._domainkey.<domain>` | TXT | DKIM public key |
| `_dmarc.<domain>` | TXT | DMARC policy |

## Consequences

**Positive:** Self-hosted, modern protocols, low resources, all-in-one
**Negative:** Email deliverability management, DNS complexity

## Related

- [SPEC-EMAIL-CONFIGURATION](./SPEC-EMAIL-CONFIGURATION.md)
