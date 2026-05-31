## 1. IMMEDIATE ACTIONS (Do This First)
### A. Block the Most Aggressive IPs at Firewall Level
```bash
# Block top attacking IPs using iptables
sudo iptables -A INPUT -s 45.134.225.130 -j DROP
sudo iptables -A INPUT -s 169.150.201.134 -j DROP
sudo iptables -A INPUT -s 110.235.232.172 -j DROP
sudo iptables -A INPUT -s 106.210.180.179 -j DROP
sudo iptables -A INPUT -s 91.224.92.45 -j DROP

# Save iptables rules (Ubuntu/Debian)
sudo iptables-save > /etc/iptables/rules.v4

# Make rules persistent
Sudo apt-get install iptables-persistent
```

B. Block Tor Network Range (Major Source of Attacks)
```bash
# Block the entire Tor exit node range that was attacking
sudo iptables -A INPUT -s 185.220.101.0/24 -j DROP
sudo iptables -A INPUT -s 185.220.100.0/24 -j DROP
```
## 2. APACHE/NGINX LEVEL PROTECTION
### A. Apache .htaccess Rules (Add to /var/www/html/.htaccess)
```bash
# Block specific malicious IPs
<RequireAll>
    Require all granted
    Require not ip 45.134.225.130
    Require not ip 169.150.201.134
    Require not ip 110.235.232.172
    Require not ip 106.210.180.179
    Require not ip 91.224.92.45
</RequireAll>

# Block fake user agents (like the misspelled "Mozlila")
RewriteEngine On
RewriteCond %{HTTP_USER_AGENT} "Mozlila" [NC]
RewriteRule .* - [F,L]

# Block directory browsing attempts
RewriteCond %{THE_REQUEST} \s/wp-includes/[^\s]*\s [NC]
RewriteRule ^wp-includes/ - [F,L]

# Rate limiting - max 60 requests per minute per IP
RewriteEngine On
RewriteMap requests_per_minute prg:/usr/local/bin/rate_limit.sh
RewriteCond ${requests_per_minute:%{REMOTE_ADDR}} >60
RewriteRule .* - [F,L]

# Block common attack patterns
RewriteCond %{QUERY_STRING} (\.\.\/|\.\.\\) [NC,OR]
RewriteCond %{QUERY_STRING} (wp-config\.php|wp-admin|xmlrpc\.php) [NC,OR]
RewriteCond %{QUERY_STRING} (eval\(|base64_decode|gzinflate) [NC]
RewriteRule .* - [F,L]
```
### B. Nginx Configuration (Add to server block)
```bash
# Block malicious IPs
location / {
    deny 45.134.225.130;
    deny 169.150.201.134;
    deny 110.235.232.172;
    deny 106.210.180.179;
    deny 91.224.92.45;
    deny 185.220.101.0/24;
    deny 185.220.100.0/24;
}

# Rate limiting
limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;
limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;

location /wp-login.php {
    limit_req zone=login burst=5 nodelay;
}

location / {
    limit_req zone=general burst=20 nodelay;
}

# Block directory browsing
location ~* /wp-includes/ {
    deny all;
    return 403;
}

# Block fake user agents
if ($http_user_agent ~* "Mozlila|Bulid") {
    return 403;
}
```
## 3. FAIL2BAN CONFIGURATION (Automated IP Blocking)
### A. Install and Configure Fail2Ban
```bash
# Install Fail2Ban
sudo apt-get update
sudo apt-get install fail2ban

# Create WordPress specific jail
sudo nano /etc/fail2ban/jail.d/wordpress.conf
```
### B. WordPress Fail2Ban Configuration
```bash
[wordpress-auth]
enabled = true
filter = wordpress-auth
logpath = /var/log/apache2/access.log
maxretry = 3
findtime = 300
bantime = 3600
action = iptables[name=wordpress-auth, port=http,protocol=tcp]

[wordpress-scan]
enabled = true
filter = wordpress-scan
logpath = /var/log/apache2/access.log
maxretry = 10
findtime = 60
bantime = 86400
action = iptables[name=wordpress-scan, port=http,protocol=tcp]

[wordpress-xmlrpc]
enabled = true
filter = wordpress-xmlrpc
logpath = /var/log/apache2/access.log
maxretry = 5
findtime = 300
bantime = 7200
action = iptables[name=wordpress-xmlrpc, port=http,protocol=tcp]
```
### C. Create Fail2Ban Filters
```bash
# WordPress authentication filter
sudo nano /etc/fail2ban/filter.d/wordpress-auth.conf
```
```bash
[Definition]
failregex = ^<HOST> .* "POST /wp-login\.php
            ^<HOST> .* "POST /wp-admin/admin-ajax\.php.*action=login
ignoreregex =
```
```bash
# WordPress scanning filter
sudo nano /etc/fail2ban/filter.d/wordpress-scan.conf
```
```bash
[Definition]
failregex = ^<HOST> .* "GET /wp-includes/
            ^<HOST> .* "GET /wp-content/.*\.php
            ^<HOST> .* "GET /wp-admin/.*" 404
            ^<HOST> .* "GET /\.env
            ^<HOST> .* "GET /\.git/config
ignoreregex =
```

### D. Start Fail2Ban
```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
sudo fail2ban-client status
```

## 4. WORDPRESS SECURITY HARDENING
### A. Update WordPress Security Plugins
```bash
# Update Wordfence (already installed)
wp plugin update wordfence --path=/var/www/html/

# Install additional security plugins
wp plugin install sucuri-scanner --activate --path=/var/www/html/
wp plugin install limit-login-attempts-reloaded --activate --path=/var/www/html/
```
### B. Disable Directory Browsing
```bash
# Add to WordPress .htaccess
echo "Options -Indexes" >> /var/www/html/.htaccess

# Secure wp-config.php
chmod 600 /var/www/html/wp-config.php
```
### C. Hide WordPress Version and Sensitive Files
```bash
// Add to wp-config.php
define('DISALLOW_FILE_EDIT', true);
define('DISALLOW_FILE_MODS', true);

// Add to functions.php
remove_action('wp_head', 'wp_generator');
```

## 5. CLOUDFLARE PROTECTION (Recommended)
### A. Setup Cloudflare
- Sign up for Cloudflare (free plan available)
- Add your domain to Cloudflare
- Update nameservers at your domain registrar
- Enable security features:
    - DDos Protection :ON
    - Web Application Firewall (WAF): ON
    - Rate Limiting: Configure rules
    - Bot Fight Mode: ON

### B. Cloudflare Security Rules
```bash
// Block countries with high attack rates
(ip.geoip.country in {"CN" "RU" "KP" "IR"})

// Block known bad user agents
(http.user_agent contains "Mozlila" or http.user_agent contains "Bulid")

// Rate limit wp-login attempts
(http.request.uri.path eq "/wp-login.php" and rate(5m) > 5)

// Block wp-includes directory access
(http.request.uri.path matches "^/wp-includes/.*")
```
## 6. MONITORING AND ALERTING
### A. Setup Log Monitoring Script
```bash
# Create monitoring script
sudo nano /usr/local/bin/attack_monitor.sh
```
```bash
#!/bin/bash
# Attack monitoring script

LOG_FILE="/var/log/apache2/access.log"
ALERT_THRESHOLD=100
TEMP_FILE="/tmp/attack_count.tmp"

# Count requests in last hour
CURRENT_HOUR=$(date +"%d/%b/%Y:%H")
ATTACK_COUNT=$(grep "$CURRENT_HOUR" "$LOG_FILE" | wc -l)

echo "$ATTACK_COUNT" > "$TEMP_FILE"

if [ "$ATTACK_COUNT" -gt "$ALERT_THRESHOLD" ]; then
    # Send alert (configure email)
    echo "High attack volume detected: $ATTACK_COUNT requests in last hour" | \
    mail -s "Security Alert - High Traffic" admin@yourdomain.com
    
    # Log to syslog
    logger "SECURITY ALERT: $ATTACK_COUNT requests detected in last hour"
fi
```

```bash
# Make executable and add to cron
sudo chmod +x /usr/local/bin/attack_monitor.sh
sudo crontab -e
# Add: 0 * * * * /usr/local/bin/attack_monitor.sh
```
## B. Setup Real-time Monitoring
```bash
# Install and configure logwatch
sudo apt-get install logwatch
sudo nano /etc/logwatch/conf/logwatch.conf
```

## 7. AUTOMATED IP BLOCKING SCRIPT
### A. Create Auto-Block Script
```bash
sudo nano /usr/local/bin/auto_block_ips.sh
```

```bash
#!/bin/bash
# Automatic IP blocking based on attack patterns

LOG_FILE="/var/log/apache2/access.log"
BLOCK_THRESHOLD=50
TEMP_DIR="/tmp/security"
mkdir -p "$TEMP_DIR"

# Get current date
CURRENT_DATE=$(date +"%d/%b/%Y")

# Find IPs with high request counts
grep "$CURRENT_DATE" "$LOG_FILE" | \
awk '{print $1}' | \
sort | uniq -c | \
awk -v threshold="$BLOCK_THRESHOLD" '$1 > threshold {print $2}' > "$TEMP_DIR/block_ips.txt"

# Block each IP
while read -r ip; do
    # Check if already blocked
    if ! iptables -L INPUT -n | grep -q "$ip"; then
        echo "Blocking IP: $ip"
        iptables -A INPUT -s "$ip" -j DROP
        logger "AUTO-BLOCKED IP: $ip for excessive requests"
    fi
done < "$TEMP_DIR/block_ips.txt"

# Save iptables rules
iptables-save > /etc/iptables/rules.v4
```
```bash
# Make executable and schedule
sudo chmod +x /usr/local/bin/auto_block_ips.sh
sudo crontab -e
# Add: */15 * * * * /usr/local/bin/auto_block_ips.sh
```

## 8. BACKUP AND RECOVERY
### A. Automated Backups
```bash
# Create backup script
sudo nano /usr/local/bin/wordpress_backup.sh
```
```bash
#!/bin/bash
BACKUP_DIR="/home/ubuntu/backups"
DATE=$(date +%Y%m%d_%H%M%S)
SITE_DIR="/var/www/html"
DB_NAME="wordpress"
DB_USER="wp_user"
DB_PASS="your_password"

mkdir -p "$BACKUP_DIR"

# Backup files
tar -czf "$BACKUP_DIR/wordpress_files_$DATE.tar.gz" -C "$SITE_DIR" .

# Backup database
mysqldump -u "$DB_USER" -p"$DB_PASS" "$DB_NAME" > "$BACKUP_DIR/wordpress_db_$DATE.sql"

# Keep only last 7 days of backups
find "$BACKUP_DIR" -name "wordpress_*" -mtime +7 -delete

echo "Backup completed: $DATE"
```

## 9. TESTING AND VALIDATION
### A. Test Security Measures
```bash
# Test fail2ban status
sudo fail2ban-client status wordpress-auth

# Test iptables rules
sudo iptables -L INPUT -n | grep DROP

# Test rate limiting
curl -I http://yourdomain.com/wp-login.php
```
### B. Security Scan
```bash
# Install security scanner
sudo apt-get install nikto
nikto -h http://yourdomain.com

# WordPress specific security check
wp plugin install wordfence --activate --path=/var/www/html/
wp eval "echo WP_CLI\Utils\launch_editor_for_input('Run Wordfence scan');" --path=/var/www/html/
```
## 10. MAINTENANCE SCHEDULE
### Daily Tasks:
- Review fail2ban logs: sudo fail2ban-client status
- Check blocked IPs: ``sudo iptables -L INPUT -n | grep DROP | wc -l``
-  Monitor attack logs: ``tail -f /var/log/apache2/access.log | grep -E "(404|403|500)"``

### Weekly Tasks:
- Update Wordpress Core and Plugins
- Review and analyze attack patterns
- Update IP block lists
- Test backup restoration

### Monthly Tasks:
- Security audit and penetration testing
- Review and update security rules
- Analyze Attack trends and adjust defenses
- Update fail2ban filters based on new attack patterns

## 11. EMERGENCY RESPONSE PLAN
### If Under Active Attack:
- Immediate Response:
  ```bash
  # Enable maintenance mode
  wp maintenance-mode activate --path=/var/www/html/
  
  # Block all traffic except your IP
  iptables -A INPUT -s YOUR_IP_ADDRESS -j ACCEPT
  iptables -A INPUT -j DROP
  ```

- Analysis:
    ```bash
    # Identify attack source
    tail -n 1000 /var/log/apache2/access.log | awk '{print $1}' | sort | uniq -c | sort -nr
    
    # Block top attackers
    # Use the auto-block script above
    ```
- Recovery:
  ```bash
  # Restore from backup if needed
  # Update security rules
  # Gradually restore normal traffic
  ```

## CONCLUSION
### This comprehensive security solution addresses:
- Immediate threat blocking (firewall level)
- Application level protection (Apache/Nginx)
- Automated threat detection (Fail2Ban)
- WordPress hardening
- CDN protection (Cloudflare)
- Monitoring and alerting
- Automated response
- Backup and recovery

### Priority Implementation Order:
- Firewall blocking (Section 1)
- Fail2Ban setup (Section 3)
- WordPress hardening (Section 4)
- Cloudflare setup (Section 5)
- Monitoring scripts (Section 6)
- Automated blocking (Section 7)

### Estimated Implementation Time: 2-4 hours Maintenance Time: 30 minutes per week

### Remember to test each component in a staging environment first and keep backups before making changes.
