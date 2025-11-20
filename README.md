# Post-Incident Report: CyberPanel / OpenLiteSpeed Failure

**Date:** 19 November 2025  
**Prepared by:** Areez Afsar Khan

---

## 1. Summary

After a CyberPanel update (v2.4.4), OpenLiteSpeed (OLS) became unstable due to a corrupted `httpd_config.conf`.  
The corruption caused OLS’s worker process to fail before binding ports 80/443, leading to repeated restarts and full website downtime.  
Restoring a clean OLS configuration file and remapping listeners was required to bring services back online.

---

## 2. Timeline of Events

### 2.1 Initial Issue
- Sites began showing **404 errors**.
- After attempting to repair vHost settings, websites changed to **“Site can't be reached.”**

### 2.2 Early Symptoms
OLS logs showed:
- `No listener is available, stop server!`
- Autorestarter loops
- PHP LSAPI child limit warnings  
**Note:** WebAdmin panel (port 7080) remained operational.

### 2.3 Debugging Actions
- Checked vHost configs → Valid  
- Verified `.htaccess` → Valid  
- Confirmed PHP 7.4 handler exists  
- Verified DNS & server IP  
- Checked firewall → Not active  
- Checked ports → Nothing listening on 80/443  
- Verified listeners in WebAdmin → Present but non-functional

### 2.4 Discovery of Root Cause
- OLS worker process was **dropping privileges before binding ports**.
- Root cause: Corrupted `httpd_config.conf` generated during CyberPanel update.

This caused:
- Failure to bind ports <1024  
- Immediate OLS worker crash  
- Only Admin console (7080) remained functional  

---

## 3. Root Cause

### Primary Cause
CyberPanel update modified/replaced `httpd_config.conf` incorrectly, causing:
- Missing or broken listener definitions  
- Incorrect privilege drop order  
- Failure to bind 80/443  
- Infinite restart loop  

### Secondary Causes
- Missing vHost mappings for affected domains  
- PHP LSAPI pool too small → additional warnings  

---

## 4. Impact

| Item | Impact |
|------|--------|
| Sites affected | Multiple domains under OLS |
| Downtime | Full HTTP(S) outage |
| Admin Panel | Operational (7080) |
| SEO/Uptime | Temporary outage |
| Users | Unable to access pages |

---

## 5. Recommended Actions (Which I didn't take yet, please test on you dev environment)

### 5.1 Validation Checks
- Verified listeners in WebAdmin  
- Ran port diagnostics (`ss -ltnp`)  
- Analyzed config structure  
- Checked logs for privilege/bind failures  
- Confirmed no port conflicts  

---

## 6. Resolution (yet to test)

The issue may be resolved by:
- Restoring a valid OLS configuration  
- Rebuilding listener-to-vHost mappings  
- Adjusting PHP LSAPI process limits  
- Restarting OLS to reinitialize the worker  

Service MAY restore once listeners successfully bound to ports 80/443 taking the following actions:

### 6.1 Recovery Actions
- Identify broken `httpd_config.conf`  
- Test config using:
  ```bash
  openlitespeed -t -f httpd_config.conf
  ```
- Restore last working config:
  ```bash
  cp httpd_config.conf.backup-20251117-1826 httpd_config.conf
  ```
- Restart OLS  
- Re-add listener mappings  
- Increased PHP LSAPI limits (`LSAPI_CHILDREN=30`, `maxConns=30`)  
- Verify ports were open and websites functional  


---

## 7. Prevention & Recommendations

### 7.1 Pre-update Backups
Backup:
- `/usr/local/lsws/conf`
- `/etc/cyberpanel`

### 7.2 Config Monitoring
Enable integrity alerts for:
- `httpd_config.conf`
- CyberPanel templates

### 7.3 Post-update Tests
Run:
```bash
openlitespeed -t -f /usr/local/lsws/conf/httpd_config.conf
ss -ltnp | egrep ':80|:443'
```

### 7.4 Resource Tuning
- Increase LSAPI children for busy WordPress sites  
- Ensure server RAM is adequate  

### 7.5 Monitoring
Use:
- UptimeRobot or BetterUptime  
- Netdata or Prometheus exporters  

---

## 8. Final Notes

This incident shows:
- CyberPanel updates may silently corrupt OLS configs  
- WebAdmin may appear normal despite backend config issues  
- Listener/port checks are essential for diagnosing OLS failures  
