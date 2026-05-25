# NGINX Plus Upgrade Summary (R32 → R37)

## 1. Compatibility & Pre-Checks

### A. OS Compatibility Check
* Verify OS version compatibility for R37.
* **Reference:** [NGINX Plus R37 Release Notes](https://docs.nginx.com/nginx/releases/#r37.0)

### B. NIM Compatibility Check
Verify the nginx compatibility with currently used NIM, if managed by NGINX Instance Manager:
* **Supported NGINX Versions:** [NIM NGINX Version Specs](https://docs.nginx.com/nginx-instance-manager/fundamentals/tech-specs/#nginx-versions)

---

## 2. Backup
* **Action:** Back up the existing NGINX configuration files before proceeding with the installation.

---

## 3. Installation Steps
Refer to the following guides for the upgrade execution:
* **Upgrade Guide (R32 and earlier):** [Upgrading NGINX Plus](https://docs.nginx.com/nginx/admin-guide/installing-nginx/upgrading-nginx-plus/#nginx-plus-r32-and-earlier)
* **Lab Reference:** [Upgrade NGINX Plus on VM](https://docs.nginx.com/nginx-one-console/workshops/lab5/upgrade-nginx-plus-to-latest-version/#exercise-b5-upgrade-nginx-plus-on-your-vm)
* **Detailed Step Reference:** [Installing NGINX Plus / Creating Directory](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-plus/#upgrading-nginx-plus:~:text=Subscription%20Licenses.-,Create%20the%20/etc/nginx/,-directory%20for%20Linux)

---

## 4. Breaking Changes

### 4.1 Mandatory JWT License
> ⚠️ **CRITICAL:** Starting with NGINX Plus R33, a JWT (JSON Web Token) license is mandatory. Without it, NGINX Plus will not start after the upgrade.

### 4.2 Changelog
* Review intermediate behavior changes and updates here: [NGINX Plus Releases Changelog](https://docs.nginx.com/nginx/releases/)

### 5. RollBack Plan:

You can list and downgrade the nginx version. Example with yum.

```bash
yum list --showduplicates nginx-plus
yum downgrade nginx-plus-<previous-version>
```

