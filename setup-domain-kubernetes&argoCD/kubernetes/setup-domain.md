Perfect! We can include your **ClusterIssuer definitions** directly in the documentation for completeness. Here’s the **updated Kubernetes Dashboard doc**, now including the **staging and production ClusterIssuer YAML** step.

---

# **Kubernetes Dashboard Setup & Domain Access**

## **1️⃣ Ensure Service Type is ClusterIP (Production)**

The Dashboard service should stay as **ClusterIP** (default). NodePort is **not needed** when using Ingress.

```bash
kubectl edit service kubernetes-dashboard -n kube-system
```

* Ensure it shows:

```yaml
spec:
  type: ClusterIP
```

* Save and exit.

* Verify:

```bash
kubectl get svc -n kube-system | grep kubernetes-dashboard
```

You should see something like:

```
kubernetes-dashboard   ClusterIP   10.233.xxx.xxx   443/TCP
```

---

## **2️⃣ Create ServiceAccount for Dashboard Login**

Create a file `kubernetes-dashboard-token.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kube-system
```

Apply it:

```bash
kubectl apply -f kubernetes-dashboard-token.yaml
```

---

## **3️⃣ Get Login Token**

```bash
kubectl -n kube-system create token admin-user
```

* Copy the token.
* You’ll use this token to log in to the Dashboard.

---

## **4️⃣ Apply ClusterIssuers (TLS Certificates)**

Make sure **cert-manager** is installed.
Create the **staging and production ClusterIssuers**:

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
    email: alexkgm2412@gmail.com
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

## **5️⃣ Create Ingress for Domain Access (ClusterIP + HTTPS)**

Create a file `kubernetes-dashboard-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubernetes-dashboard-ingress
  namespace: kube-system
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  rules:
    - host: dashboard.cambostack.codes     # Replace with your domain
      http:
        paths:
          - path: "/"
            pathType: Prefix
            backend:
              service:
                name: kubernetes-dashboard
                port:
                  number: 443
  tls:
    - hosts:
        - dashboard.cambostack.codes
      secretName: kubernetes-dashboard-tls
```

Apply it:

```bash
kubectl apply -f kubernetes-dashboard-ingress.yaml
```

Verify:

```bash
kubectl get ingress -n kube-system
```

---

## **6️⃣ Access Dashboard via Domain**

Open in browser:

```
https://dashboard.cambostack.codes
```

* Paste the token from Step 3 to log in.

> The Ingress handles HTTPS via cert-manager automatically. No NodePort or `kubectl proxy` is needed.

---

This version now includes **both staging and production ClusterIssuers**, making it fully production-ready and aligned with best practices.

---

If you want, I can **also make the ArgoCD documentation match this style**, including staging/prod ClusterIssuers and ClusterIP setup.

Do you want me to do that?
