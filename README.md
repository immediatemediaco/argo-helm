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
   - CSI is the only supported source of the Redis password.

4. **`argocd-secret` still required**  
   - Argo CD’s core credentials (admin password hash, session signing key) are still stored in the Kubernetes Secret `argocd-secret`.  
   - You may delete the bootstrap `argocd-initial-admin-secret` after first login, but `argocd-secret` must remain (unless you patch upstream).

---

## Required Helm Values

```yaml
redis:
  passwordFromFile:
    enabled: true
    path: /mnt/secrets/redis-password/_k8s_argocd_redis_password
#argocd-server pod
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
#argocd-application-controller pod
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
#argocd-repo-server pod
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

### Admin Credentials

- The API server reads the hashed admin password and signing key from the `argocd-secret` Kubernetes Secret.
- You may safely delete the bootstrap `argocd-initial-admin-secret` after your first login.
- **Do not delete or modify `argocd-secret`**—removing its data will break admin login and core Argo CD functionality.
- Only the admin password hash and session signing key are stored here; Redis credentials are handled via CSI as described above.
Deleting only argocd-initial-admin-secret after first login is safe; deleting data from argocd-secret breaks the admin login.
- Hashed admin password will be deleted once the sso has been configured for user management (there is a plan to implement this feature)

### Quick Reference

| Pod                         | Needs CSI volume                        | Requires command override           |
|-----------------------------|-----------------------------------------|-------------------------------------|
| argocd-application-controller | ✅                                      | ✅                                   |
| argocd-repo-server            | ✅                                      | ✅                                   |
| argocd-server                 | ✅                                      | ✅                                   |
| Redis (argocd-redis)          | ✅ (reads the file in its entrypoint)   | No additional command needed         |

If any of the command lists or volumes are missing, expect CrashLoopBackOff or connection errors.
```
