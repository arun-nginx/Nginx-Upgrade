# NGINX Plus Upgrade Reference

**Disclaimer: This is not official documentation and should be used only as a reference guide. Please validate and test all steps before implementation, and make any necessary modifications based on your environment. It is recommended to confirm the steps with the F5 Support team for the latest updates and validation.**

> **Upgrade Paths:** R32 → R37 (VM) | R35 → R37 (Containers)

## ⚠️ CRITICAL: Breaking Change Alert

**R32 to R33+ is a BREAKING CHANGE**

Starting with NGINX Plus R33, a **JWT (JSON Web Token) license is mandatory**. Without it, NGINX Plus will not start after upgrade.

- **VM Path (R32→R37):** This is the #1 blocker. You must obtain and install the JWT before upgrading.
- **Container Path (R35→R37):** JWT is already in use, so this does not apply.

---

## 📖 Table of Contents

- [Part 1: VM Upgrade (R32 → R37)](#-part-1-vm-upgrade--r32--r37)
  - [Phase 1: Pre-Checks](#-phase-1-pre-checks-before-any-upgrade-activity)
  - [Phase 2: Upgrade Steps](#-phase-2-upgrade-steps-vm--r32--r37)
  - [Phase 3: Post-Upgrade Validation](#-phase-3-post-upgrade-validation-vm)
- [Part 2: Container Upgrade (R35 → R37)](#-part-2-container-upgrade--r35--r37)
  - [Phase 1: Pre-Checks](#-phase-1-pre-checks-containers)
  - [Phase 2: Upgrade Steps](#-phase-2-upgrade-steps-containers--r35--r37)
  - [Phase 3: Post-Upgrade Validation](#-phase-3-post-upgrade-validation-containers)
- [Summary Comparison](#-summary-comparison-table)
- [References](#-key-references)

---

## 📋 PART 1: VM Upgrade — R32 → R37

### 🔍 Phase 1: Pre-Checks (Before Any Upgrade Activity)

#### 1.1 Inventory & Environment Checks

```bash
# Check current NGINX Plus version
nginx -v

# Check OS version (verify R37 support)
cat /etc/os-release

# Check currently installed modules
nginx -V 2>&1 | grep -o '\-\-with[^ ]*'

# Check current config syntax
sudo nginx -t

# Check NGINX service status
sudo systemctl status nginx

# Check if NGINX Agent is installed (if managed by NIM)
sudo systemctl status nginx-agent
```

#### 1.2 OS Compatibility Check for R37
Review Latest Doc : https://github.com/arun-nginx/Nginx-Upgrade/blob/main/Upgrade.md

> ⚠️ **Important:** Ubuntu 20.04 is removed as of R35. If running Ubuntu 20.04, an OS upgrade is required first.

#### 1.3 JWT License — MANDATORY for R32→R33+ Upgrade

This is the **#1 blocker** for your upgrade. Without this, the upgrade will fail with:

```
!!! NGINX Plus was not upgraded !!!
NGINX Plus R33 introduces a breaking change.
```

This is referenced from this doc, so you can review this document as well to crosscheck for any doubts: 
https://docs.nginx.com/nginx/admin-guide/installing-nginx/upgrading-nginx-plus/#nginx-plus-r32-and-earlier

**Steps to obtain and place the JWT:**

1. Log in to [MyF5 Portal](https://account.f5.com/myf5)
2. Go to **My Products & Plans** > **Subscriptions**
3. Find your NGINX subscription → Select **Subscription ID**
4. Download the **JSON Web Token (JWT)** file
5. Place it on each NGINX Plus instance:

```bash
sudo mkdir -p /etc/nginx
sudo cp license.jwt /etc/nginx/license.jwt
sudo chmod 644 /etc/nginx/license.jwt
```

6. Validate the JWT has no trailing spaces/newlines:

```bash
xxd /etc/nginx/license.jwt | tail -3
# Ensure no "20" (space) or "0a" (newline) at end of file
```

> 📌 **Note:** Per [KB K000148865](https://my.f5.com/manage/s/article/K000148865), the JWT must be in the same directory as `nginx.conf` (default: `/etc/nginx/license.jwt`). If using a custom `nginx.conf` path, place it in that same directory.

#### 1.4 Network / Connectivity Check (R33+ Requirement)

R33+ requires outbound HTTPS (TCP 443) for usage reporting.

**Option A: Internet-connected**
- Allow outbound to `product.connect.nginx.com` on port 443

**Option B: Disconnected/Air-gapped**
- Configure NIM as the usage reporting endpoint:

```nginx
# Add to nginx.conf main block (AFTER upgrade)
mgmt {
    usage_report endpoint=<NIM-FQDN>;
}
```

> ⚠️ **Warning:** Per [KB K000153170](https://my.f5.com/manage/s/article/K000153170), failure to configure usage reporting in disconnected environments will result in 503 errors and NGINX stopping traffic processing.


#### 1.5 NIM Compatibility Check (If Managed by NIM)

**Check Documentation:**
**Supported Linux Distributions:** https://docs.nginx.com/nginx-instance-manager/fundamentals/tech-specs/

**Supported NGINX Instance Manager versions:** https://docs.nginx.com/nginx-instance-manager/fundamentals/tech-specs/#supported-nginx-instance-manager-versions

**Supported NGINX Versions:** https://docs.nginx.com/nginx-instance-manager/fundamentals/tech-specs/#nginx-versions

> 📌 **Requirement:** NGINX Plus R33+ requires **NGINX Instance Manager 2.18 or later**.

```bash
# Check NIM version if applicable
sudo systemctl status nms
```

#### 1.6 Backup Configurations

```bash
# Backup NGINX config and logs
sudo cp -a /etc/nginx /etc/nginx-backup-$(date +%Y%m%d)
sudo cp -a /var/log/nginx /var/log/nginx-backup-$(date +%Y%m%d)

# Backup SSL certs if stored under /etc/nginx
sudo cp -a /etc/ssl/nginx /etc/ssl/nginx-backup-$(date +%Y%m%d)
```
### 🔧 Phase 2: Upgrade Steps (VM — R32 → R37)

> 📌 **Note:** Per [KB K000138256](https://my.f5.com/manage/s/article/K000138256), direct multi-version upgrades are supported, but you must review the "Important Changes in Behavior" for each intermediate release (R33, R34, R35, R36, R37).

#### Step 1: Place JWT (if not done in pre-checks)

```bash
sudo cp license.jwt /etc/nginx/license.jwt
```

#### Step 2: Update the NGINX Plus Repository

**For Debian/Ubuntu:**

```bash
# Refresh repo
sudo apt-get update

# Upgrade NGINX Plus to latest (R37)
sudo apt-get install -y nginx-plus

# If upgrading specific modules (e.g., App Protect), upgrade them too
sudo apt-get install -y nginx-plus-module-appprotect
```

**For RHEL/CentOS/Oracle/Rocky/AlmaLinux:**

```bash
# Refresh repo
sudo yum clean all

# Upgrade NGINX Plus to latest (R37)
sudo yum update -y nginx-plus

# If upgrading specific modules
sudo yum update -y nginx-plus-module-appprotect
```

#### Step 3: Verify the Upgrade

```bash
# Check new version
nginx -v

# Test configuration syntax
sudo nginx -t
```

#### Step 4: Add mgmt Block (for R33+ — if not already present)

If upgrading from R32, add the `mgmt` block to `nginx.conf` for usage reporting:

```nginx
# For internet-connected environments (add to main block of nginx.conf)
mgmt {
    # No additional config needed for direct internet reporting
}

# For disconnected/NIM-managed environments
mgmt {
    usage_report endpoint=<YOUR-NIM-FQDN>;
}
```

#### Step 5: Reload/Restart NGINX

```bash
# Test config first
sudo nginx -t

# Reload (zero-downtime)
sudo nginx -s reload

# OR restart if needed
sudo systemctl restart nginx
```

#### Step 6: Verify Usage Reporting (R33+ Compliance)

```bash
# Set log level to info temporarily
sudo vi /etc/nginx/nginx.conf
# Set: error_log /var/log/nginx/error.log info;

sudo nginx -s reload

# Monitor for usage report confirmation
sudo tail -f /var/log/nginx/error.log | grep "usage report"
# Look for: [info] ... usage report was sent
```

> 📌 **Reference:** [KB K000153173](https://my.f5.com/manage/s/article/K000153173)

### ✅ Phase 3: Post-Upgrade Validation (VM)

```bash
# 1. Confirm version
nginx -v

# 2. Confirm service is running
sudo systemctl status nginx

# 3. Confirm config is valid
sudo nginx -t

# 4. Confirm JWT is valid (no invalid license token errors)
sudo tail -50 /var/log/nginx/error.log | grep -i "license\|jwt\|usage"

# 5. Confirm traffic is flowing (check access logs)
sudo tail -f /var/log/nginx/access.log

# 6. If NIM-managed, verify instance shows online in NIM dashboard
sudo systemctl status nginx-agent
```
---

## 📋 PART 2: Container Upgrade — R35 → R37

The container upgrade path is significantly simpler than the VM path:

- ✅ JWT is already in use (R35 already requires it)
- ✅ No in-place package upgrade — containers are replaced with new images
- ✅ Recommended approach: **Blue/Green deployment** (new container alongside old)

### 🔍 Phase 1: Pre-Checks (Containers)

#### 1.1 Verify Current Container State

```bash
# List running NGINX Plus containers
docker ps | grep nginx-plus

# Check current image/version
docker inspect <container-name> | grep -i "image\|version"

# Check current NGINX version inside container
docker exec <container-name> nginx -v
```

#### 1.2 Verify JWT is Available

```bash
# Confirm JWT is accessible (env var or mounted volume)
docker exec <container-name> cat /etc/nginx/license.jwt
# OR check env var
docker inspect <container-name> | grep -i "JWT\|LICENSE"
```

#### 1.3 Pull and Verify the R37 Image

```bash
# Login to NGINX private registry using JWT
export JWT=$(cat /path/to/nginx-repo.jwt)
echo "$JWT" | docker login private-registry.nginx.com \
  --username "$JWT" --password-stdin

# List available tags to confirm R37 exists
curl https://private-registry.nginx.com/v2/nginx-plus/agent/tags/list \
  --key /etc/ssl/nginx/nginx-repo.key \
  --cert /etc/ssl/nginx/nginx-repo.crt | jq | grep r37

# Pull the R37 image (example for Debian)
docker pull private-registry.nginx.com/nginx-plus/agent:r37-debian
```

#### 1.4 Review Release Notes for R36 and R37

Check for any breaking changes between R35 and R37 at [NGINX Plus Releases](https://docs.nginx.com/nginx/releases/).

### 🔧 Phase 2: Upgrade Steps (Containers — R35 → R37)

The recommended approach is **deploy new → validate → retire old** (blue/green):

#### Step 1: Pull the R37 Image

```bash
docker pull private-registry.nginx.com/nginx-plus/agent:r37-debian
# OR tag for your private registry
docker tag private-registry.nginx.com/nginx-plus/agent:r37-debian \
  <my-registry>/nginx-plus/agent:r37
docker push <my-registry>/nginx-plus/agent:r37
```

#### Step 2: Deploy New R37 Container (Blue/Green)

```bash
# Launch new R37 container with same config volumes and JWT
sudo docker run \
  --env=NGINX_AGENT_SERVER_GRPCPORT=443 \
  --env=NGINX_AGENT_SERVER_HOST=<NIM-HOST> \
  --env=NGINX_AGENT_TLS_ENABLE=true \
  --env=NGINX_LICENSE_JWT=$JWT \
  --restart=always \
  --runtime=runc \
  -v /path/to/nginx.conf:/etc/nginx/nginx.conf \
  -v /path/to/conf.d:/etc/nginx/conf.d \
  -p 80:80 -p 443:443 \
  -d <my-registry>/nginx-plus/agent:r37
```

**For Docker Compose**, update the image tag in `compose.yaml` and run:

```bash
docker compose up --force-recreate -d
```

#### Step 3: Validate the New Container

```bash
# Check version inside new container
docker exec <new-container-name> nginx -v

# Check config syntax
docker exec <new-container-name> nginx -t

# Check logs for errors
docker logs <new-container-name> | tail -50

# Verify usage reporting
docker exec <new-container-name> tail -f /var/log/nginx/error.log | grep "usage report"
```

#### Step 4: Shift Traffic to New Container

1. Update your load balancer or DNS to point to the new R37 container
2. Monitor traffic and error rates

#### Step 5: Retire Old R35 Container

```bash
# Stop and remove old container
docker stop <old-r35-container>
docker rm <old-r35-container>

# Clean up old image (optional)
docker rmi private-registry.nginx.com/nginx-plus/agent:r35-debian
```

#### Step 6: Clean Up Stale Entries in NIM (if applicable)

If using NIM/NGINX One Console, remove unavailable (old) instances:

1. Go to **Instances** → Filter by **Availability = Unavailable**
2. Select old instances → **Delete selected**

> 📌 **Reference:** Per Lab 5: Upgrade NGINX Plus to the latest version

### ✅ Phase 3: Post-Upgrade Validation (Containers)

```bash
# 1. Confirm new version
docker exec <container-name> nginx -v

# 2. Confirm container is healthy
docker ps | grep nginx-plus
docker inspect <container-name> | grep -i "status\|health"

# 3. Confirm config is valid
docker exec <container-name> nginx -t

# 4. Confirm no license/JWT errors
docker logs <container-name> | grep -i "license\|jwt\|usage\|error"

# 5. Confirm usage reporting is working
docker exec <container-name> tail -20 /var/log/nginx/error.log | grep "usage report"

# 6. Confirm traffic is flowing
docker logs <container-name> | grep -v "^$" | tail -20
```
---

## 📊 Summary Comparison Table (Validate below with support or PS)

| Item | VM: R32 → R37 | Container: R35 → R37 |
|------|--------------|---------------------|
| **JWT Required?** | ✅ YES — Critical blocker | ✅ Already in use |
| **Breaking Change?** | ✅ YES (R33 licensing change) | ❌ No visible breaking changes but still validate with support |
| **Upgrade Method** | In-place package upgrade | Replace container image |
| **Downtime Risk** | Low (reload) | Minimal (blue/green) |
| **Backup Required?** | ✅ YES | Config in volumes/CM |
| **NIM Version Check?** | ✅ YES (≥ 2.18 for R33+) | ✅ YES if NIM-managed |
| **Usage Reporting Config** | Required in nginx.conf | Via env var or config |
| **OS Compatibility** | Must verify per distro | N/A (image-based) |

---
## 📚 Key References

### Knowledge Base Articles

- [KB K000148681](https://my.f5.com/manage/s/article/K000148681): NGINX Plus was NOT Upgraded — JWT missing error
- [KB K000148865](https://my.f5.com/manage/s/article/K000148865): JWT License Location — JWT file placement
- [KB K000148775](https://my.f5.com/manage/s/article/K000148775): Invalid License Token — Trailing chars in JWT
- [KB K000153170](https://my.f5.com/manage/s/article/K000153170): 503 After Upgrade in Disconnected Env — mgmt block config
- [KB K000153173](https://my.f5.com/manage/s/article/K000153173): How to View Usage Report — Verify reporting
- [KB K000159973](https://my.f5.com/manage/s/article/K000159973): AWS Marketplace Upgrade — AWS-specific note
- [KB K000138256](https://my.f5.com/manage/s/article/K000138256): Multi-version upgrade support

### Documentation & Resources

- [NGINX Plus Releases](https://docs.nginx.com/nginx/releases/) — Release dates & EoSD policy
- [Lab 5: Upgrade NGINX Plus](https://clouddocs.f5.com/training/community/nginx/html/class9/module2/lab5.html) — Docker & VM upgrade lab
- [MyF5 Portal](https://account.f5.com/myf5) — Download JWT and manage subscriptions

---

**Document Version:** 1.0  
**Last Updated:** 2026  
**Maintained By:** Arun Gopakumar
