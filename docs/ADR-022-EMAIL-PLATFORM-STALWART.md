# ADR-022: Email Platform - Stalwart Mail Server

## Status
**Accepted**

## Date
2026-01-07

## Context

Talent Mesh requires email capabilities for:
- Notifications (assessment reminders, results)
- User communication
- System alerts

Options considered:

1. **External SaaS**: SendGrid, AWS SES, Mailgun
2. **Self-hosted traditional**: Postfix + Dovecot + Rspamd
3. **Self-hosted modern**: Stalwart, WildDuck, Mox, Maddy

Key constraints:
- **Cost**: Avoid recurring SaaS fees for a new organization
- **Integration**: Use existing infrastructure (PostgreSQL, Dragonfly)
- **K8s native**: Must run well in Kubernetes
- **Modern protocols**: JMAP preferred over legacy IMAP

## Decision

We will use **Stalwart Mail Server** with **Roundcube** for webmail.

### Why Stalwart

| Factor | Stalwart | WildDuck | Mox | SendGrid |
|--------|----------|----------|-----|----------|
| **Language** | Rust | Node.js | Go | N/A |
| **Memory** | **~200MB** | ~500MB-1GB | ~100-300MB | N/A |
| **K8s native** | ✅ Native clustering | ⚠️ Docker only | ❌ Single binary | N/A |
| **Database** | **PostgreSQL** ✅ | MongoDB | SQLite | N/A |
| **Coordination** | **Dragonfly/Redis** ✅ | MongoDB | N/A | N/A |
| **JMAP** | ✅ Yes | ❌ No | ❌ No | ❌ No |
| **CalDAV/CardDAV** | ✅ Yes | ❌ No | ❌ No | ❌ No |
| **OIDC/OAuth** | ✅ Yes | ❌ No | ❌ No | N/A |
| **Spam filter** | ✅ LLM-powered | External | Bayesian | N/A |
| **Cost** | **$0** | $0 | $0 | ~$15+/month |

### Architecture

```mermaid
flowchart TB
    subgraph clients["Email Clients"]
        webmail["Roundcube<br/>(Browser)"]
        mobile["Mobile Apps<br/>(Outlook, K-9)"]
        desktop["Desktop<br/>(Thunderbird)"]
    end

    subgraph stalwart_cluster["Stalwart Cluster (3 replicas)"]
        s1["Stalwart 1<br/>~200MB"]
        s2["Stalwart 2<br/>~200MB"]
        s3["Stalwart 3<br/>~200MB"]
    end

    subgraph protocols["Protocols"]
        jmap["JMAP<br/>(modern API)"]
        imap["IMAP4rev2<br/>(port 993)"]
        smtp["SMTP<br/>(port 587)"]
        caldav["CalDAV/CardDAV"]
    end

    subgraph storage["Storage Backend"]
        pg["PostgreSQL<br/>(CNPG)"]
        dragonfly["Dragonfly<br/>(coordination)"]
        minio["MinIO<br/>(attachments)"]
    end

    subgraph security["Security"]
        spam["LLM Spam Filter"]
        dkim["DKIM/SPF/DMARC"]
    end

    webmail --> jmap
    mobile --> imap & smtp
    desktop --> imap & smtp & caldav

    jmap & imap & smtp & caldav --> stalwart_cluster
    s1 & s2 & s3 --> pg
    s1 & s2 & s3 --> dragonfly
    s1 & s2 & s3 --> minio
    s1 & s2 & s3 --> spam & dkim
```

### Protocol Support

| Protocol | Port | Use Case |
|----------|------|----------|
| SMTP (submission) | 587 | Sending email (STARTTLS) |
| SMTPS | 465 | Sending email (implicit TLS) |
| IMAP4rev2 | 993 | Receiving email (TLS) |
| JMAP | 443 | Modern API (HTTP/JSON) |
| CalDAV | 443 | Calendar sync |
| CardDAV | 443 | Contact sync |
| ManageSieve | 4190 | Server-side filtering |

### Stalwart Configuration

```yaml
# stalwart.yaml
server:
  hostname: mail.talentmesh.io

cluster:
  node-id: 1
  peers:
    - stalwart-1.stalwart:7911
    - stalwart-2.stalwart:7911
    - stalwart-3.stalwart:7911
  coordination:
    type: redis
    url: redis://dragonfly:6379

storage:
  data:
    type: postgresql
    url: postgres://stalwart:${DB_PASSWORD}@postgres-cluster-rw:5432/stalwart
  blob:
    type: s3
    bucket: stalwart-blobs
    endpoint: http://minio:9000
    access-key: ${MINIO_ACCESS_KEY}
    secret-key: ${MINIO_SECRET_KEY}

authentication:
  fallback-admin:
    user: admin
    secret: ${ADMIN_SECRET}

  # OIDC integration (LinkedIn OAuth)
  oidc:
    - id: linkedin
      issuer: https://api.talentmesh.io
      client-id: ${OIDC_CLIENT_ID}
      client-secret: ${OIDC_CLIENT_SECRET}

spam:
  filter:
    type: llm
    model: local  # Use local LLM for spam detection
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: stalwart
  namespace: email
spec:
  serviceName: stalwart
  replicas: 3
  selector:
    matchLabels:
      app: stalwart
  template:
    metadata:
      labels:
        app: stalwart
    spec:
      containers:
      - name: stalwart
        image: stalwartlabs/stalwart:latest
        ports:
        - containerPort: 25    # SMTP
        - containerPort: 587   # Submission
        - containerPort: 993   # IMAPS
        - containerPort: 443   # HTTPS/JMAP
        - containerPort: 7911  # Cluster
        resources:
          requests:
            memory: "200Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: stalwart-secrets
              key: db-password
        volumeMounts:
        - name: config
          mountPath: /opt/stalwart/etc
      volumes:
      - name: config
        configMap:
          name: stalwart-config
```

### Webmail: Roundcube

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: roundcube
  namespace: email
spec:
  replicas: 1
  selector:
    matchLabels:
      app: roundcube
  template:
    spec:
      containers:
      - name: roundcube
        image: roundcube/roundcubemail:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "200Mi"
            cpu: "100m"
        env:
        - name: ROUNDCUBEMAIL_DEFAULT_HOST
          value: "ssl://stalwart"
        - name: ROUNDCUBEMAIL_DEFAULT_PORT
          value: "993"
        - name: ROUNDCUBEMAIL_SMTP_SERVER
          value: "tls://stalwart"
        - name: ROUNDCUBEMAIL_SMTP_PORT
          value: "587"
```

### Mobile Device Compatibility

| Device | App | Status | Notes |
|--------|-----|--------|-------|
| iOS | **Outlook** | ✅ Recommended | Full functionality |
| iOS | Apple Mail | ⚠️ Workaround | Setup via macOS first, or use app passwords |
| iOS | Spark | ✅ Works | Standard IMAP |
| Android | **K-9 Mail** | ✅ Recommended | Full IMAP support |
| Android | Gmail | ✅ Works | Add as "Other" account |
| Android | Outlook | ✅ Works | Full functionality |

### DNS Records Required

```
; MX record
talentmesh.io.    IN MX 10 mail.talentmesh.io.

; A record
mail.talentmesh.io.    IN A <istio-gateway-ip>

; SPF
talentmesh.io.    IN TXT "v=spf1 mx -all"

; DKIM (auto-generated by Stalwart)
mail._domainkey.talentmesh.io.    IN TXT "v=DKIM1; k=rsa; p=..."

; DMARC
_dmarc.talentmesh.io.    IN TXT "v=DMARC1; p=quarantine; rua=mailto:dmarc@talentmesh.io"

; MTA-STS
_mta-sts.talentmesh.io.    IN TXT "v=STSv1; id=20260107"

; TLSRPT
_smtp._tls.talentmesh.io.    IN TXT "v=TLSRPTv1; rua=mailto:tlsrpt@talentmesh.io"
```

### Resource Requirements

| Component | Instances | Memory | CPU |
|-----------|-----------|--------|-----|
| Stalwart | 3 | ~200MB each | 100m each |
| Roundcube | 1 | ~200MB | 100m |
| **Total** | | **~800MB** | ~400m |

## Consequences

### Positive

1. **Uses existing infrastructure**: PostgreSQL (CNPG) + Dragonfly
2. **Modern protocols**: JMAP for efficient API access
3. **Full collaboration**: CalDAV/CardDAV included
4. **OIDC integration**: Can share auth with LinkedIn OAuth
5. **LLM spam filter**: No external spam service needed
6. **Rust efficiency**: Low memory, high performance
7. **Zero cost**: No SaaS fees

### Negative

1. **iOS Mail complexity**: Requires workaround for native app
2. **No built-in webmail**: Need separate Roundcube deployment
3. **DNS complexity**: Multiple records required for email deliverability

### Mitigations

- Document iOS Mail setup workaround clearly
- Pre-configure Roundcube with optimal settings
- Provide DNS template for quick setup
- Use cert-manager for TLS certificates

## References

- [Stalwart Mail Server](https://stalw.art/)
- [Stalwart Documentation](https://stalw.art/docs/)
- [Stalwart Clustering](https://stalw.art/docs/cluster/overview/)
- [Roundcube Webmail](https://roundcube.net/)
- [JMAP Protocol](https://jmap.io/)

---

*ADR Version: 1.0*
*Last Updated: 2026-01-07*
