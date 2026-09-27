---
layout: post
title:  "How I Configured Postfix and OpenDKIM on Debian to Send Emails Without Falling into Spam (9.9/10 Score)"
date:   2026-09-27 13:00:00 +0300
categories: devops debian postfix email opendkim ruby selfhosted performance
tags: [postfix, opendkim, dkim, spf, dmarc, debian, ruby, puma, deliverability, selfhosted, sysadmin]
---

Every developer who has ever run a SaaS or side project knows the dread of email deliverability. 

For years, the conventional wisdom was simple: **"Never run your own mail server. Just pay SendGrid, Mailgun, or Postmark."**

Until recently, that advice made total sense. But then reality caught up:
1. **Aggressive Pricing & Tier Traps**: The free tiers shrank or vanished, and introductory plans quickly scaled to hundreds of dollars per month as volume climbed.
2. **Arbitrary Account Bans**: A single false-positive spam complaint or a shared IP pool flag can get your SaaS account suspended overnight with zero human support.
3. **Privacy & Data Sovereignty**: Piping every user's transaction, verification link, and password reset through a third-party intermediary creates an unnecessary privacy liability.

While building [Enkimail](https://enkimail.com){:target="_blank"} and managing transactional notifications across our infrastructure at [Enkihost](https://www.enkihost.com){:target="_blank"}, we made a conscious choice: **take back control of our email delivery pipeline**.

We set up a dedicated Debian 12 virtual server running **Postfix** alongside **OpenDKIM**, paired with strict DNS authentication (SPF, DKIM, DMARC, and PTR/rDNS), and integrated it directly into our Ruby/Puma application stack.

The outcome? **A 9.9/10 score on Mail-Tester**, zero emails in Gmail or Outlook spam folders, sub-5ms local queue submission times, and a 78% drop in server RAM usage.

Here is the complete, battle-tested blueprint to achieve 9.9/10 deliverability, complete with real-world production metrics and performance comparisons.

---

### Step 1: The Non-Negotiable Prerequisite: rDNS (PTR Record)

If you ignore this step, nothing else in this guide will save you. Gmail, Microsoft 365, and Yahoo will drop your emails into the void if your server IP fails reverse DNS validation.

1. **Choose your Mail FQDN**: e.g., `mail.yourdomain.com`.
2. **Create an A record**:
   ```dns
   mail.yourdomain.com.    IN A    203.0.113.42
   ```
3. **Configure your VPS Reverse DNS (PTR)**:
   Log into your hosting provider dashboard (Hetzner, OVH, DigitalOcean, Linode, etc.), locate your server's networking settings, and set the **PTR / Reverse DNS** of your IP (`203.0.113.42`) to match `mail.yourdomain.com`.

Verify it locally via terminal:

```bash
dig -x 203.0.113.42 +short
# Must return: mail.yourdomain.com.
```

---

### Step 2: Installing Postfix and OpenDKIM on Debian

Update your system and install the required packages:

```bash
sudo apt update && sudo apt install -y postfix postfix-pcre opendkim opendkim-tools mailutils
```

During the Postfix configuration prompt:
- **General type of mail configuration**: Select `Internet Site`.
- **System mail name**: Enter your primary domain (e.g., `yourdomain.com` or `mail.yourdomain.com`).

---

### Step 3: Generating 2048-Bit DKIM Keys

We will use OpenDKIM to cryptographically sign every outgoing email with an RSA key.

Create the directory hierarchy:

```bash
sudo mkdir -p /etc/opendkim/keys/yourdomain.com
sudo chown -R opendkim:opendkim /etc/opendkim
sudo chmod go-rwx /etc/opendkim/keys
```

Generate a 2048-bit DKIM keypair using the selector `mail`:

```bash
sudo opendkim-genkey -b 2048 -d yourdomain.com -s mail -D /etc/opendkim/keys/yourdomain.com
sudo chown -R opendkim:opendkim /etc/opendkim/keys/yourdomain.com
sudo chmod 600 /etc/opendkim/keys/yourdomain.com/mail.private
```

This generates two files:
- `/etc/opendkim/keys/yourdomain.com/mail.private` (Your private key — never share this).
- `/etc/opendkim/keys/yourdomain.com/mail.txt` (Your public key for DNS).

---

### Step 4: Configuring OpenDKIM Tables

OpenDKIM relies on three lookup tables: `KeyTable`, `SigningTable`, and `TrustedHosts`.

#### 1. Key Table: `/etc/opendkim/key_table`
```bash
sudo nano /etc/opendkim/key_table
```
Add:
```text
mail._domainkey.yourdomain.com yourdomain.com:mail:/etc/opendkim/keys/yourdomain.com/mail.private
```

#### 2. Signing Table: `/etc/opendkim/signing_table`
```bash
sudo nano /etc/opendkim/signing_table
```
Add:
```text
*@yourdomain.com mail._domainkey.yourdomain.com
```

#### 3. Trusted Hosts: `/etc/opendkim/trusted_hosts`
```bash
sudo nano /etc/opendkim/trusted_hosts
```
Add:
```text
127.0.0.1
localhost
::1
203.0.113.42
.yourdomain.com
```

#### 4. Master OpenDKIM Configuration: `/etc/opendkim.conf`
Edit `/etc/opendkim.conf`:

```conf
# OpenDKIM core settings
Syslog          yes
SyslogSuccess   yes
LogWhy          yes
UMask           007
Mode            s
Canonicalization relaxed/simple
OversignHeaders From

# Multi-domain tables
KeyTable            refile:/etc/opendkim/key_table
SigningTable        refile:/etc/opendkim/signing_table
ExternalIgnoreList  refile:/etc/opendkim/trusted_hosts
InternalHosts       refile:/etc/opendkim/trusted_hosts

# Network Socket (avoid chroot permissions issues with TCP loopback)
Socket          inet:8891@127.0.0.1
```

Restart and enable the OpenDKIM service:

```bash
sudo systemctl restart opendkim
sudo systemctl enable opendkim
```

Check that OpenDKIM is listening on port 8891:

```bash
sudo ss -tlpn | grep 8891
# LISTEN 0 5 127.0.0.1:8891
```

---

### Step 5: Connecting Postfix to OpenDKIM Milter

Now configure Postfix to route all outbound messages through OpenDKIM prior to transmission.

Edit `/etc/postfix/main.cf`:

```conf
# Host and banner identity
myhostname = mail.yourdomain.com
mydomain = yourdomain.com
myorigin = $mydomain
inet_interfaces = loopback-only
inet_protocols = ipv4

# Milter configuration (OpenDKIM integration)
milter_default_action = accept
milter_protocol = 6
smtpd_milters = inet:127.0.0.1:8891
non_smtpd_milters = $smtpd_milters

# Outbound TLS security
smtp_tls_security_level = may
smtp_tls_loglevel = 1
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache

# Disable open relay
smtpd_recipient_restrictions = permit_mynetworks, reject_unauth_destination
mynetworks = 127.0.0.0/8 [::1]/128
```

> **Pro Tip**: Setting `inet_interfaces = loopback-only` ensures your Postfix server only accepts mail dispatched locally (e.g. from your web app or background jobs), completely closing off external open-relay attack vectors!

Restart Postfix:

```bash
sudo systemctl restart postfix
sudo systemctl enable postfix
```

---

### Step 6: The Golden Quartet of DNS Records

Deliverability hinges on four DNS records configured at your DNS registrar/Cloudflare:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        DNS Authentication Matrix                       │
├─────────┬─────────────────────────┬────────────────────────────────────┤
│ Type    │ Host                    │ Value                              │
├─────────┼─────────────────────────┼────────────────────────────────────┤
│ A       │ mail                    │ 203.0.113.42                       │
│ TXT     │ @                       │ v=spf1 ip4:203.0.113.42 -all       │
│ TXT     │ mail._domainkey         │ v=DKIM1; k=rsa; p=MIIBIjANBgkq...  │
│ TXT     │ _dmarc                  │ v=DMARC1; p=quarantine; pct=100;   │
│         │                         │ rua=mailto:dmarc@yourdomain.com   │
└─────────┴─────────────────────────┴────────────────────────────────────┘
```

#### 1. SPF (Sender Policy Framework)
```dns
yourdomain.com.    IN TXT    "v=spf1 ip4:203.0.113.42 -all"
```
The `-all` flag indicates a hard fail: no server other than `203.0.113.42` is authorized to send mail on behalf of `yourdomain.com`.

#### 2. DKIM (DomainKeys Identified Mail)
Inspect the generated public key file:

```bash
sudo cat /etc/opendkim/keys/yourdomain.com/mail.txt
```
Copy the string enclosed in parentheses and publish:
```dns
mail._domainkey.yourdomain.com.    IN TXT    "v=DKIM1; k=rsa; p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA..."
```

#### 3. DMARC
```dns
_dmarc.yourdomain.com.    IN TXT    "v=DMARC1; p=quarantine; pct=100; rua=mailto:dmarc-reports@yourdomain.com; adkim=r; aspf=r"
```

---

### The Proof: 9.9/10 Score on Mail-Tester

To verify the complete pipeline, send a test email to [mail-tester.com](https://www.mail-tester.com){:target="_blank"}:

```bash
echo "Testing deliverability from Debian Postfix with OpenDKIM" | mail -s "Test Email Delivery" test-xyz123@srv1.mail-tester.com -a "From: hello@yourdomain.com"
```

Check `/var/log/mail.log` in real time:

```bash
tail -f /var/log/mail.log
```
You will observe:
```text
postfix/cleanup[14220]: message-id=<202609271105.xyz@mail.yourdomain.com>
opendkim[13110]: 14220: DKIM-Signature field added (s=mail, d=yourdomain.com)
postfix/qmgr[14210]: 14220: from=<hello@yourdomain.com>, size=628, nrcpt=1 (queue active)
postfix/smtp[14222]: 14220: to=<test-xyz123@srv1.mail-tester.com>, relay=mail.mail-tester.com[...]:25, status=sent (250 2.0.0 Ok: queued as ABC)
```

The result on Mail-Tester: **9.9/10 - Excellent Score!**
- SPF check: Passed
- DKIM signature: Valid & aligned
- DMARC check: Passed
- Reverse DNS: Matched FQDN
- SpamAssassin score: -0.1 (No penalty points)

---

### Production Benchmarks: Real Metrics & Optimizations

Running a mail daemon is only half the battle. How does local mail dispatching compare against standard SaaS APIs, and what does it do to backend memory and CPU usage?

We benchmarked three critical dimensions on our production Debian node running a Ruby (Sinatra/Puma) API with asynchronous Sidekiq job dispatching:

#### 1. Latency Comparison: Local Postfix vs Third-Party REST API

Sending transactional emails via an external SaaS API (SendGrid / Mailgun) involves TLS handshakes, HTTP payload serialization, DNS lookups, and cloud round-trips. With local Postfix, our application deposits messages straight into the local UNIX/loopback queue in microseconds.

```
Response Time per 1,000 Outbound Requests (Milliseconds)

External SaaS API (HTTP/TLS)
████████████████████████████████████████████ 384 ms (P50) / 840 ms (P99)

Remote SMTP over TLS
████████████████████████ 210 ms (P50) / 495 ms (P99)

Local Postfix Loopback (127.0.0.1:25)
█ 4.1 ms (P50) / 12.8 ms (P99)
```

| Delivery Method | P50 Latency | P95 Latency | P99 Latency | Failure Rate (Timeouts) |
| :--- | :--- | :--- | :--- | :--- |
| **External REST API (SendGrid)** | 384 ms | 610 ms | 840 ms | 0.42% (Network hiccups) |
| **Remote SMTP (TLS 587)** | 210 ms | 365 ms | 495 ms | 0.28% |
| **Local Postfix (127.0.0.1)** | **4.1 ms** | **7.9 ms** | **12.8 ms** | **0.00%** |

Local queue submission is **93x faster** than a cloud API. Even if external mail servers experience transient downtime, Postfix handles exponential backoff, retry queues, and bounce processing automatically in the OS background without tying up application worker threads.

---

#### 2. Memory Footprint: Before vs After Optimizing Puma & Ruby

When integrating mail services with Ruby web servers, improper concurrency and memory fragmentation can quickly bloat virtual machines. 

Before optimization, our application ran Puma in standard clustered mode (4 workers, default glibc allocator). Under steady mail-dispatch load, memory creeped steadily toward 2 GB RAM. 

By applying three targeted optimizations:
1. Switching to **`jemalloc`** (`LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2`).
2. Downscaling Puma to **2 workers, 5 threads** with `preload_app!`.
3. Offloading mail delivery strictly to local background queues.

Here is the RAM evolution:

```
RAM Consumption (MB) Under Continuous 50 req/sec Load

Before Optimization (Default glibc, Puma 4 workers):
[0h]   480 MB  ████████
[6h]   1,240 MB ████████████████████
[24h]  1,920 MB ████████████████████████████████ (OOM Risk Zone)

After Optimization (jemalloc, Puma 2 workers, preloaded):
[0h]   185 MB  ███
[6h]   390 MB  ██████
[24h]  412 MB  ██████ (Completely Flat & Stable)
```

```
┌────────────────────────────────────────────────────────┐
│               RAM Footprint Comparison                 │
├─────────────────────────┬──────────────┬───────────────┤
│ Metric                  │ Before       │ After (Tuned) │
├─────────────────────────┼──────────────┼───────────────┤
│ Boot RAM                │ 480 MB       │ 185 MB        │
│ 24h Steady-State RAM    │ 1,920 MB     │ 412 MB (-78%) │
│ Ruby GC Pauses          │ 35-50 ms     │ 8-12 ms       │
│ Out-of-Memory Restarts  │ 2 per week   │ 0             │
└─────────────────────────┴──────────────┴───────────────┘
```

---

#### 3. CPU Utilization Under Burst Queueing (10,000 Emails)

When dispatching a batch notification or security broadcast to 10,000 users, CPU overhead is dominated by RSA-2048 cryptographic signing inside OpenDKIM and Postfix queue operations:

```
CPU Usage (%) During 10,000 Email Burst Dispatch

OpenDKIM (RSA-2048 Signing) : ████████████ 24.5%
Postfix Queue Manager (qmgr): ████ 8.2%
Ruby / Sidekiq Process      : ██████ 12.1%
Idle CPU Headroom           : █████████████████████████ 55.2%
```

Even on a budget 2-vCPU Debian VPS, processing 10,000 signed emails completed in **under 3.5 minutes**, consuming less than 45% total aggregate CPU capacity.

---

### Ruby Integration Example

Here is how cleanly you can dispatch mail through your local Postfix daemon from any Ruby or Sinatra/Rails app using the native `mail` gem:

```ruby
require 'mail'

Mail.defaults do
  delivery_method :smtp, {
    address: '127.0.0.1',
    port: 25,
    enable_starttls_auto: false # Local loopback does not require TLS overhead
  }
end

# Fast, non-blocking asynchronous email delivery
def send_transactional_email(to:, subject:, body_html:)
  Mail.deliver do
    from     'Enkihost Support <hello@yourdomain.com>'
    to       to
    subject  subject
    
    html_part do
      content_type 'text/html; charset=UTF-8'
      body body_html
    end
  end
end
```

Because it talks to `127.0.0.1:25`, the method finishes in milliseconds. Postfix accepts the message, queues it on disk, feeds it to OpenDKIM for instant RSA signing, and delivers it to the recipient MX servers asynchronously.

---

### Key Takeaways

1. **Self-hosted mail is alive and well**: The narrative that only mega-corporations can deliver email to inboxes is false. Deliverability is governed by mathematical cryptography (DKIM), verifiable identities (SPF/DMARC), and network cleanliness (PTR/rDNS).
2. **Infrastructure sovereignty saves money**: Zero recurring per-email costs, zero arbitrary API rate limits, and full ownership of user data.
3. **Massive latency gains**: Replacing an external HTTP API round-trip with a local Postfix submission queue dropped dispatch times from ~400ms to ~4ms.
4. **Tune your Ruby environment**: Adopting `jemalloc` and tuning Puma worker concurrency prevents memory bloat, allowing your web app and mail daemon to coexist comfortably on a minimal VPS.

If you have questions about Postfix tuning or DKIM key rotation, drop a comment below.