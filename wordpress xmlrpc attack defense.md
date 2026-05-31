# How I Mitigated 1.1 Million XML-RPC Attacks on a WordPress Server Using Apache Rate Limiting and Fail2Ban

## Introduction

Recently, I encountered a serious security incident on a production WordPress website hosted on AWS. The server was receiving more than **1.1 million malicious XML-RPC requests**, causing high memory consumption, Apache process accumulation, and repeated Out-Of-Memory (OOM) kills.

The attack was sophisticated because it originated from thousands of different IP addresses across multiple countries. Traditional firewall-based IP blocking was not an effective long-term solution.

In this article, I will explain:

- The attack pattern
- Why IP blocking failed
- How I identified the root cause
- How I implemented Apache rate limiting
- How I configured Fail2Ban
- Additional security recommendations

---

# The Problem

During routine monitoring, I noticed unusual spikes in:

- CPU utilization
- Memory usage
- Apache process count
- Network traffic

Soon after, critical services started crashing.

### Symptoms

```text
Apache service killed by OOM Killer
MariaDB service terminated
Website downtime
Slow response times
```

System logs showed repeated memory exhaustion events.

```text
01:28:59 - MariaDB OOM kill
01:31:19 - Apache2 OOM kill
02:20:40 - Another OOM event
02:22:56 - Apache2 killed again
```

---

# Investigating the Attack

After analyzing Apache access logs, I found a common pattern:

```text
POST /xmlrpc.php HTTP/1.1
```

Example requests:

```text
192.108.48.150 - POST /xmlrpc.php
23.191.200.32 - POST /xmlrpc.php
194.15.36.117 - POST /xmlrpc.php
78.142.18.172 - POST /xmlrpc.php
89.58.26.216 - POST /xmlrpc.php
```

Further analysis revealed:

- More than 1,108,462 XML-RPC requests
- Continuous attack activity
- Multiple countries involved
- Fake browser user agents
- Spoofed Google referrers
- Distributed botnet behavior

---

# Initial Approach: Firewall IP Blocking

My first instinct was to block attacking IP addresses using firewall rules.

### Using UFW

```bash
sudo ufw deny from 192.108.48.150
```

### Using iptables

```bash
sudo iptables -A INPUT -s 192.108.48.150 -j DROP
```

Initially, this appeared effective.

However, after monitoring for several hours, a major problem became obvious.

Every attack request originated from a different IP address.

Example:

```text
192.108.48.150
23.191.200.32
194.15.36.117
78.142.18.172
89.58.26.216
...
Thousands more
```

This was a distributed botnet attack.

### Why IP Blocking Failed

- Attackers continuously rotated IPs.
- New IPs appeared every few seconds.
- Maintaining large firewall blocklists was not scalable.
- Server resources were still consumed before detection.

At this point, it became clear that IP-based blocking was not a permanent solution.

---

# Second Approach: Apache Rate Limiting

Instead of blocking individual IPs, I decided to limit how many requests a client could make.

The objective was simple:

> Allow normal users, but throttle abusive behavior.

---

# Installing Apache Security Modules

## Enable Required Modules

```bash
sudo a2enmod ratelimit
sudo systemctl restart apache2
```

---

# Installing mod_evasive

Apache's mod_evasive module helps detect and block excessive requests automatically.

```bash
sudo apt update
sudo apt install libapache2-mod-evasive
```

---

# Configuring mod_evasive

Create configuration:

```bash
sudo nano /etc/apache2/mods-enabled/evasive.conf
```

Configuration:

```apache
<IfModule mod_evasive20.c>

DOSHashTableSize 3097

DOSPageCount 5
DOSPageInterval 10

DOSSiteCount 100
DOSSiteInterval 10

DOSBlockingPeriod 600

</IfModule>
```

### Explanation

| Directive | Purpose |
|------------|----------|
| DOSPageCount | Maximum requests per page |
| DOSPageInterval | Time window |
| DOSSiteCount | Maximum site-wide requests |
| DOSBlockingPeriod | Block duration |

Result:

```text
More than 5 requests
to the same URL
within 10 seconds

= Block for 10 minutes
```

---

# Protecting XML-RPC Endpoint

Since the attack targeted XML-RPC specifically, I added additional protection.

### Apache Configuration

```apache
<Location "/xmlrpc.php">

SetOutputFilter RATE_LIMIT
SetEnv rate-limit 400

</Location>
```

This reduced abuse while still allowing legitimate traffic if XML-RPC was required.

---

# Implementing Fail2Ban

Rate limiting alone was not enough.

I wanted repeat offenders to be automatically banned.

---

## Install Fail2Ban

```bash
sudo apt install fail2ban -y
```

Verify:

```bash
sudo systemctl status fail2ban
```

---

# Creating Apache XML-RPC Filter

Create:

```bash
sudo nano /etc/fail2ban/filter.d/apache-xmlrpc.conf
```

Add:

```ini
[Definition]

failregex = ^<HOST> .*POST /xmlrpc\.php.*

ignoreregex =
```

---

# Creating Jail Configuration

```bash
sudo nano /etc/fail2ban/jail.local
```

Add:

```ini
[apache-xmlrpc]

enabled = true
port = http,https
filter = apache-xmlrpc

logpath = /var/log/apache2/access.log

maxretry = 10
findtime = 300
bantime = 3600
```

---

# How It Works

Fail2Ban monitors:

```text
/var/log/apache2/access.log
```

If an IP performs:

```text
10 XML-RPC requests
within 5 minutes
```

Fail2Ban automatically executes:

```bash
iptables -A INPUT -s <IP> -j DROP
```

The IP is banned for:

```text
1 hour
```

---

# Results After Deployment

After implementing:

- Apache Rate Limiting
- mod_evasive
- Fail2Ban

I observed significant improvements.

### Before

```text
Memory Usage: 90%+
Apache OOM Kills
MariaDB OOM Kills
Website Downtime
```

### After

```text
Stable Memory Usage
Reduced Apache Processes
No OOM Events
Improved Availability
Lower Resource Consumption
```

---

# Additional Security Recommendations

For production WordPress environments, I strongly recommend:

### Disable XML-RPC Completely (If Not Needed)

```apache
<Files xmlrpc.php>
Require all denied
</Files>
```

### Use Cloudflare WAF

Benefits:

- DDoS protection
- Bot mitigation
- Global CDN
- Rate limiting

### Enable Fail2Ban

Protects:

- SSH
- WordPress Login
- XML-RPC
- Apache Abuse

### Keep WordPress Updated

Always update:

- WordPress Core
- Themes
- Plugins

---

# Key Lessons Learned

The biggest lesson from this incident was:

> Firewall IP blocking works for small attacks but fails against distributed botnets.

When attackers rotate thousands of IP addresses, maintaining blocklists becomes impossible.

A better approach is:

1. Rate limit abusive traffic.
2. Automatically ban repeat offenders.
3. Use a WAF/CDN layer.
4. Reduce attack surface.

Combining Apache Rate Limiting and Fail2Ban provided a practical and scalable defense against ongoing XML-RPC attacks.

---

# Conclusion

WordPress XML-RPC attacks remain one of the most common attack vectors on public websites.

In this case, over 1.1 million malicious requests caused severe resource exhaustion and repeated service outages.

While firewall-based IP blocking was my first attempt, it quickly became clear that a distributed botnet required a more intelligent solution.

By implementing Apache rate limiting and Fail2Ban, I was able to significantly reduce malicious traffic, improve server stability, and prevent future outages.

For any DevOps engineer managing WordPress infrastructure, these tools should be part of your baseline security strategy.

---

**Tags:** #DevOps #WordPress #Apache #Fail2Ban #CyberSecurity #AWS #Linux #XMLRPC #RateLimiting #WebSecurity
