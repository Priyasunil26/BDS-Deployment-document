## Sentry Web Pod Issue: **no app loaded. GAME OVER**

### **Issue**

The container logs showed the following error:

```
unable to load app 0 (mountpoint='') (callable not found or import error)
*** no app loaded. GAME OVER ***
```

The pod remained in `Running` state but failed to properly serve the application.

---

### **Cause**

This issue occurs because the **Sentry database migrations were not applied** after the deployment. Without these migrations:

* Required database tables and configurations are missing.
* The application cannot fully initialize.
* uWSGI is unable to load the Django app, resulting in the "no app loaded" error.

---

### ✅ **Resolution**

Running the following command inside the pod resolves the issue:

```bash
kubectl exec -it sentry-web-7d5657dd6f-w844h -n dev-sentry -- sentry upgrade
```

This command:

* Applies pending database migrations.
* Sets up required internal Sentry data.
* Allows the web pod to fully initialize and start serving requests.
