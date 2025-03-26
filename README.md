

# Kubernetes Ingress-NGINX Upgrade & CVE-2025-1974 Mitigation Guide
## Purpose
This guide walks you through upgrading the Ingress-NGINX Controller in your Kubernetes cluster to the latest version v1.12.1, and mitigating the ```CVE-2025-1974``` security vulnerability related to unsafe annotation snippets.


Note: ```Helm Link```: https://github.com/kubernetes/ingress-nginx/releases


## 1. Upgrade to Ingress-NGINX v1.12.1
### Apply the latest official manifest:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.1/deploy/static/provider/cloud/deploy.yaml
```

This will update the controller and deploy the necessary resources (Deployment, Service, Admission Webhooks, etc.).


## 2. Fix Immutable Field Error (admission job)
### If you see an error like this:

```
"The Job "ingress-nginx-admission-create" is invalid: field is immutable"
```
It means a job already exists and Kubernetes doesn't allow changing immutable fields in jobs.

### Fix:

```
kubectl delete job ingress-nginx-admission-create -n ingress-nginx
kubectl delete job ingress-nginx-admission-patch -n ingress-nginx
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.1/deploy/static/provider/cloud/deploy.yaml
```

## 3. Verify the Upgrade

Check if pods are running:

```
kubectl get pods -n ingress-nginx
```

Check controller image version:

```
kubectl describe deployment ingress-nginx-controller -n ingress-nginx | grep Image
```
Expected output:
```
Image:  registry.k8s.io/ingress-nginx/controller:v1.12.1
```

## 4. Disable Dangerous Annotation Snippets
To prevent users from injecting unsafe NGINX config (which led to the vulnerability), ensure the following flag is set in your controller Deployment:

### Edit the deployment:

```
kubectl edit deployment ingress-nginx-controller -n ingress-nginx
```
Add this under args::

```
- --allow-snippet-annotations=false
```

This disables annotations like:


- nginx.ingress.kubernetes.io/server-snippet

- configuration-snippet

- proxy-snippet


## 5. Fix Your Ingress YAML (Avoid 404 Errors)

Ensure this field exists too:

```
spec:
  ingressClassName: nginx
```
Without ingressClassName, the new controller won’t handle the resource, resulting in 404 Not Found.

- Optional secure annotation sample of ingress

```
annotations:
  kubernetes.io/ingress.class: 'nginx'
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
  nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
  nginx.ingress.kubernetes.io/server-tokens: "false"
  nginx.ingress.kubernetes.io/proxy-body-size: "5m"
  nginx.ingress.kubernetes.io/proxy-hide-headers: "Server"
```

## 6. Test Your Application Access

Access your domain via browser.

### Run:

```
kubectl get ingress -n your-namespace
kubectl describe ingress your-ingress-name -n your-namespace
```

Check that routes, services, and TLS are all working properly.


## 7. Alternative: How to Add Security Headers Safely
If you're removing configuration-snippet but still want to add headers like HSTS or X-Content-Type-Options, consider these options:

Best Practice:
Set headers inside your backend application.

Optional (advanced):
Use the add-headers directive in the Ingress-NGINX ConfigMap to apply global headers to all responses.

Let me know if you'd like help setting that up.

