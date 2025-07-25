# Alloi On-Premises Deployment
## Step 1: Install Dependencies
### What are we doing in this step?
We are preparing our Kubernetes cluster so that Alloi can work properly.
Think of Kubernetes as a big school, and we are adding some important staff members:

cert-manager – A teacher who gives out "certificates" (SSL certificates) so that your website can be accessed securely using https://.

NGINX ingress controller – A gatekeeper who controls which class (service) a visitor should go to when they enter the school.

## Commands Explanation
## 1. Install cert-manager
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml
```
### What this does:
Downloads the cert-manager setup file from the internet and tells Kubernetes to install it.
This will allow our cluster to create SSL certificates (like a lock on your website).




