# NGINX Plus Container Upgrade Summary (R35 → R37)

## 1. Compatibility & Pre-Checks

### A. Environment & Release Analysis
* Verify current active container state, environment variables, and active image version using `docker ps` and `docker inspect`.
* Review intermediate changes and behavioral differences between R35, R36, and R37.
* **Reference:** [NGINX Plus Releases](https://docs.nginx.com/nginx/releases/)

### B. NIM & NGINX One Compatibility Check
Verify container compatibility with your management plane if managed by NGINX Instance Manager (NIM) or NGINX One Console:
* **Supported NGINX Versions:** [NIM NGINX Version Specs](https://docs.nginx.com/nginx-instance-manager/fundamentals/tech-specs/#nginx-versions)

---

## 2. Backup
* **Action:** Back up existing configurations (`nginx.conf`, `conf.d`), certificates, and persistent logs on the host machine before deploying the new container image.

---

## 3. Installation Steps (Blue/Green Strategy)
Refer to the following guides for container upgrade orchestration and image deployment:
* **Lab Reference:** [Upgrade NGINX Plus on Docker](https://docs.nginx.com/nginx-one-console/workshops/lab5/upgrade-nginx-plus-to-latest-version/)
* **Execution:** Log into the private registry (`private-registry.nginx.com`) using your JWT, pull the target `r37` image, spin up the new container alongside the old one, test syntax via `nginx -t`, shift traffic, and safely terminate the R35 container.

---

## 4. Breaking Changes & License Compliance

### 4.1 JWT License Status
> 💡 **NOTE:** Because NGINX Plus R35 already utilizes the JSON Web Token (JWT) architecture, the mandatory license breaking changes introduced in R33 do not apply to this path. Ensure your `$JWT` remains properly mounted or injected as an environment variable.

### 4.2 Disconnected Environments & Usage Reporting
> ⚠️ **CRITICAL:** R33+ requires outbound HTTPS (TCP 443) connectivity to `product.connect.nginx.com` for usage reporting. In air-gapped or disconnected environments, you must configure a `mgmt {}` block pointing to NIM to avoid 503 errors.

---

## 5. Rollback Plan

If issues arise during validation or traffic shifting, the fallback process is instantaneous due to the containerized environment.

```bash
# 1. Re-route traffic back to the old R35 container via your load balancer/DNS
# 2. If the old container was already stopped, re-launch it using the R35 image:
docker start <old-r35-container>

# 3. Stop and debug the failed R37 container
docker stop <new-r37-container>
