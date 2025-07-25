# Alloi On-Premises Deployment
## Step 1: Install Dependencies
### What are we doing in this step?
We are preparing our Kubernetes cluster so that Alloi can work properly.
Think of Kubernetes as a big school, and we are adding some important staff members:

cert-manager – A teacher who gives out "certificates" (SSL certificates) so that your website can be accessed securely using https://.

NGINX ingress controller – A gatekeeper who controls which class (service) a visitor should go to when they enter the school.

## Commands Explanation
# Step-1
## 1. Install cert-manager
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml
```
### What this does:
Downloads the cert-manager setup file from the internet and tells Kubernetes to install it.
This will allow our cluster to create SSL certificates (like a lock on your website).

## 2. Install NGINX ingress controller
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
```
### What this does:
Downloads and installs the NGINX ingress controller.
NGINX ingress works like a smart traffic controller:
When someone visits alloi.yourdomain.com, it decides which Alloi service should answer.

## 3. Wait for cert-manager services to be ready
```
kubectl wait --for=condition=available --timeout=300s deployment -n cert-manager --all
```
### What this does:
It waits until cert-manager is fully started and ready.
The --timeout=300s means wait for up to 5 minutes.

## 4. Wait for NGINX ingress pods to be ready
```
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=ingress-nginx -n ingress-nginx --timeout=300s
```
### What this does:
It waits until NGINX ingress controller pods (the traffic controllers) are ready and working.

### In simple words:
We are installing the people who give certificates (cert-manager).
We are installing the school gatekeeper (NGINX ingress).
We wait until both are ready before moving forward.


# Step-2



