# Purpose of This Fork

This repository is a fork of the official ArgoCD Helm chart, created with the primary goal of enabling CSI Secret Store integration **without syncing secrets to Kubernetes Secrets**.

By default, the upstream ArgoCD Helm chart supports CSI integration but still mirrors secrets (such as Redis passwords, admin credentials, etc.) into Kubernetes Secrets for internal use.  
For security and compliance reasons, this fork removes all Kubernetes secret generation and modifies ArgoCD configurations to consume credentials directly from mounted CSI files instead.

## Key Objectives

- **Remove Kubernetes Secret Syncing:**  
  Disable creation and syncing of sensitive credentials (e.g., `argocd-secret`, `argocd-redis`, `argocd-initial-admin-secret`) to Kubernetes Secrets.

- **Enable File-based Secret Consumption:**  
  Modify ArgoCD Helm configurations and container startup logic to read passwords (Redis, admin, etc.) directly from files mounted by CSI drivers.

- **Integrate with AWS Secrets Manager via CSI:**  
  Allow ArgoCD components (such as `argocd-server`, `argocd-repo-server`, and `argocd-redis`) to securely consume secrets from AWS Secrets Manager using the AWS Secrets Store CSI Driver.

## Implementation Highlights

- Added support for mounting secrets from CSI volumes under `/mnt/secrets-store/` paths.
- Adjusted ArgoCD component startup commands to read Redis password and admin credentials from mounted files instead of Kubernetes secrets.
- Removed or disabled Helm templates responsible for generating Kubernetes Secret resources.
- Updated values and environment variable references to use file-based secret paths.
- Compatible with AWS Secrets Manager CSI Driver and other CSI providers.

> **Note:**  
> This fork is maintained internally for secure deployments of ArgoCD where Kubernetes Secrets are not permitted due to organizational or compliance policies.  
> Ensure that the corresponding AWS Secrets Manager entries and CSI volume mounts are correctly configured before deployment.  
> The chart behavior otherwise remains aligned with the official upstream ArgoCD Helm chart.

---

# Argo CD Redis Password (CSI-only) Customization

This fork removes all inline Redis-password handling from the upstream chart.  
The Argo CD controller, repo-server, and api-server pods now read the password from a CSI-mounted file at runtime.

> **Important**  
> These pods will crash if the CSI volume or wrapper command is missing.

---

## Summary of Changes

1. **Redis password**  
   - `redis.passwordFromFile.enabled` must be `true`.  
   - The file path (e.g. `/mnt/secrets/redis-password/_k8s_argocd_redis_password`) is injected into every pod that needs Redis.

2. **Explicit wrapper commands**  
   - Each pod—`argocd-application-controller`, `argocd-repo-server`, and `argocd-server`—now requires a `*.passwordFromFile.command` in values.  
   - The chart no longer generates a fallback script.

3. **Inline secret-init removed**  
   - The old `redis.auth.password`/`redisSecretInit` flows no longer populate the pods.  
   - CSI is the only supported source of the Redis password now.

4. **`argocd-secret` still required**  
   - Argo CD’s core credentials (admin password hash, session signing key) are still stored in the Kubernetes Secret `argocd-secret`.  
   - You can delete the bootstrap `argocd-initial-admin-secret` after first login if the pod is recreated, but `argocd-secret` must remain, until SSO has been configured.

## Required Helm Values

```yaml
redis:
  passwordFromFile:
    enabled: true
    path: /mnt/secrets/redis-password/_k8s_argocd_redis_password
  server:
    volumes:
      - name: argocd-redis-password
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: argocd-redis-password
    volumeMounts:
      - name: argocd-redis-password
        mountPath: /mnt/secrets/redis-password
        readOnly: true
    passwordFromFile:
      command:
        - sh
        - -c
        - |
            PASSWORD_FILE="/mnt/secrets/redis-password/_k8s_argocd_redis_password"
            if [ ! -f "$PASSWORD_FILE" ]; then
              echo "Redis password file $PASSWORD_FILE not found" >&2
              exit 1
            fi
            export REDIS_PASSWORD="$(cat "$PASSWORD_FILE")"
            exec /usr/local/bin/argocd-server
  controller:
    volumes:
      - name: argocd-redis-password
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: argocd-redis-password
    volumeMounts:
      - name: argocd-redis-password
        mountPath: /mnt/secrets/redis-password
        readOnly: true
    passwordFromFile:
      command:
        - sh
        - -c
        - |
            PASSWORD_FILE="/mnt/secrets/redis-password/_k8s_argocd_redis_password"
            if [ ! -f "$PASSWORD_FILE" ]; then
              echo "Redis password file $PASSWORD_FILE not found" >&2
              exit 1
            fi
            export REDIS_PASSWORD="$(cat "$PASSWORD_FILE")"
            exec /usr/local/bin/argocd-application-controller
  repoServer:
    volumes:
      - name: argocd-redis-password
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: argocd-redis-password
    volumeMounts:
      - name: argocd-redis-password
        mountPath: /mnt/secrets/redis-password
        readOnly: true
    passwordFromFile:
      command:
        - sh
        - -c
        - |
            PASSWORD_FILE="/mnt/secrets/redis-password/_k8s_argocd_redis_password"
            if [ ! -f "$PASSWORD_FILE" ]; then
              echo "Redis password file $PASSWORD_FILE not found" >&2
              exit 1
            fi
            export REDIS_PASSWORD="$(cat "$PASSWORD_FILE")"
            exec /usr/local/bin/argocd-repo-server
```

### Admin Credentials

- The API server reads the hashed admin password and signing key from the `argocd-secret` Kubernetes Secret.
- You may safely delete the bootstrap `argocd-initial-admin-secret` after your first login.
- **Do not delete or modify `argocd-secret`**—removing its data will break admin login and core Argo CD functionality.
- Only the admin password hash and session signing key are stored here; Redis credentials are handled via CSI as described above.
- Hashed admin password will be deleted once SSO has been configured for user management (planned feature).

### Quick Reference

| Pod                           | Needs CSI volume                        | Requires command override           |
|-------------------------------|-----------------------------------------|-------------------------------------|
| argocd-application-controller | ✅                                      | ✅                                   |
| argocd-repo-server            | ✅                                      | ✅                                   |
| argocd-server                 | ✅                                      | ✅                                   |
| Redis (argocd-redis)          | ✅ (reads the file in its entrypoint)   | No additional command needed         |

If any of the command lists or volumes are missing, expect CrashLoopBackOff or connection errors.

