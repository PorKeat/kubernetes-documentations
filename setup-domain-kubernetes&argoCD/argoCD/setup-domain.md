Perfect! Let’s **update your ArgoCD documentation** to include **staging/production ClusterIssuers**, just like we did for Kubernetes Dashboard. I’ll also make it production-ready, using **ClusterIP + Ingress + HTTPS**.

---

# **ArgoCD Setup & Domain Access**

## **1️⃣ Ensure ArgoCD Service is ClusterIP (Production)**

The ArgoCD server service should stay as **ClusterIP**. NodePort is **not needed** when using Ingress.

```bash
kubectl get svc -n argocd
```

You should see:

```
argocd-server   ClusterIP   10.xxx.xxx.xxx   443/TCP
```

If it is NodePort, edit the service to change back to ClusterIP:

```bash
kubectl edit svc argocd-server -n argocd
# set spec.type to ClusterIP
```

---

## **2️⃣ (Optional) Setup Admin Credentials**

The default admin password is stored in a secret:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

* Copy this password to log in to the ArgoCD UI.

---

## **3️⃣ Apply ClusterIssuers (TLS Certificates)**

Make sure **cert-manager** is installed. Apply **staging and production ClusterIssuers**:

```yaml
# letsencrypt-prod
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # Production server (trusted certs, rate-limited)
    server: https://acme-v02.api.letsencrypt.org/directory
    email: lyvanna544@gmail.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          ingress:
            class: nginx

---
# letsencrypt-staging
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # Staging server (no rate limits, untrusted certs)
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: lyvanna544@gmail.com
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

Apply them:

```bash
kubectl apply -f clusterissuer/
```

> Use the staging issuer for testing, then production issuer for live domain.

---

## **4️⃣ Create Ingress for Domain Access (ClusterIP + HTTPS)**

Create a file `argo-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  rules:
    - host: argocd.cambostack.codes     # Replace with your ArgoCD domain
      http:
        paths:
          - path: "/"
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 443
  tls:
    - hosts:
        - argocd.cambostack.codes
      secretName: argocd-tls
```

Apply it:

```bash
kubectl apply -f argoCD/argo-ingress.yaml
```

Verify:

```bash
kubectl get ingress -n argocd
```

---

## **5️⃣ Configure DNS**

Point your domain to your cluster’s public IP:

```
argocd.cambostack.codes → <Cluster Public IP>
```

---

## **6️⃣ Access ArgoCD UI**

Open in browser:

```
https://argocd.cambostack.codes
```

* Login with the default admin user and the password from Step 2.
* You can later change the admin password or configure SSO/OAuth.

---

✅ **Notes:**

* Service remains **ClusterIP**; NodePort is **not required**.
* Ingress + cert-manager handles external access and HTTPS.
* Use **staging ClusterIssuer** first for testing before switching to production.

---

If you want, I can now **give you a full project folder structure** with all your YAMLs for **both Kubernetes Dashboard and ArgoCD**, ready to deploy with ClusterIP + Ingress + TLS.

Do you want me to do that?
