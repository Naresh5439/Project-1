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



# Step-2 : Create SSL Certificate Issuer
```
cat << EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@domain.com  # Replace with your email
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```
### What is happening in this step?
We want our website (Alloi) to open using https:// (secure website).
To do that, we need an SSL certificate.
Let's Encrypt is a free service that gives us SSL certificates.
cert-manager (installed in Step 1) is like a helper who will ask Let's Encrypt for the SSL certificate whenever we need it.

### What does the command do?
The big command you see is creating something called a ClusterIssuer.
Think of a ClusterIssuer as a "rule card" that tells cert-manager:
"Whenever someone needs an SSL certificate, ask Let's Encrypt, and use my email to register."

### In super simple words:
We are telling Kubernetes: “I want to use Let's Encrypt to create SSL certificates.”
We give it our email and some settings.
Next time we ask for HTTPS, it will do the work automatically.


# Step 3: Set Up PostgreSQL Databases
### What are we doing here?
PostgreSQL is a database — like a big notebook where Alloi stores all its data.
Alloi needs 4 separate databases (4 different notebooks) to keep its information organized.
Each database has its own user (like a person with a key to open that notebook).
We are creating these databases and giving the right users permission to access them.

### 1. Create the Databases
```
CREATE DATABASE "alloi-embeddings";
CREATE DATABASE "alloi-jackson";
CREATE DATABASE "alloi-supertokens";
CREATE DATABASE "alloi-backend";
```
### These commands create 4 new databases:
alloi-embeddings – for storing AI embeddings data.
alloi-jackson – for internal Alloi data.
alloi-supertokens – for authentication (login) data.
alloi-backend – for Alloi backend application data.

### 2. Create the Users
```
CREATE USER embeddings_user WITH PASSWORD 'secure_password_1';
CREATE USER jackson_user WITH PASSWORD 'secure_password_2';
CREATE USER supertokens_user WITH PASSWORD 'secure_password_3';
CREATE USER backend_user WITH PASSWORD 'secure_password_4';
```
### We are creating 4 users (like 4 people, each with a password) so each one can access their own database.
Example: embeddings_user can open and write in alloi-embeddings.

### 3. Give Permissions
```
GRANT ALL PRIVILEGES ON DATABASE "alloi-embeddings" TO embeddings_user;
GRANT ALL PRIVILEGES ON DATABASE "alloi-jackson" TO jackson_user;
GRANT ALL PRIVILEGES ON DATABASE "alloi-supertokens" TO supertokens_user;
GRANT ALL PRIVILEGES ON DATABASE "alloi-backend" TO backend_user;
```
### Here we are saying:
"Each user can fully control their own database."
Without this step, they would not have permission to read/write data.

### In super simple words:
We are creating 4 notebooks (databases), 4 people with keys (users), and telling each person, "You can write and read your own notebook."


# Step 4: Clone and Prepare Alloi Charts
### What are we doing in this step?
We are downloading the Alloi Helm charts (these are like ready-made instructions to install Alloi on Kubernetes).
We are updating any extra files (dependencies) needed by the charts.
We are creating a namespace in Kubernetes called alloi (like creating a separate classroom just for Alloi).

### 1. Clone the repository
```
git clone --branch INFRA-11-restructure-alloi-stack-and-dependencies https://github.com/opshealth/alloi-public-charts.git
```
### What this does:
Downloads the Alloi public charts from GitHub (like copying a folder from the internet onto your computer).
This folder contains all the files needed to install Alloi.

### 2. Move into the folder
```
cd alloi-public-charts
```
### What this does:
Changes the current working directory to alloi-public-charts so we can work inside it.

### 3. Update chart dependencies
```
helm dependency update ./alloi-stack
```
### What this does:
Helm charts often rely on other charts (extra pieces).
This command downloads and updates those extra pieces so everything is ready for installation.

Think of it like:
Before cooking a meal, you check and download all missing ingredients.

### 4. Create a namespace for Alloi
```
kubectl create namespace alloi
```
### What this does:
Creates a special "container" (namespace) in Kubernetes called alloi.
All Alloi services (pods, deployments, etc.) will live inside this namespace.

### In simple words:
We are downloading Alloi's installation files, updating them, and preparing a clean room (namespace) where we will set everything up.


# Step 5: Configure DNS
### What are we doing in this step?
When you want to visit Alloi using a domain like alloi.yourdomain.com, your computer needs to know where that website is hosted.
DNS (Domain Name System) is like the phonebook of the internet — it converts the domain name into an IP address (like a phone number).

We will:
Find the IP address (or hostname) of the load balancer created by Kubernetes.
Create a DNS record that connects alloi.yourdomain.com to that IP.

### 1. Get your Load Balancer IP/hostname
```
kubectl get svc -n ingress-nginx ingress-nginx-controller
```
### What this does:
It checks the ingress-nginx namespace for the ingress-nginx-controller service.
This service is like the main gate through which people access Alloi.

The output will show something like:
```
NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)
ingress-nginx-controller   LoadBalancer   10.0.1.5        34.120.45.67     80:32224/TCP
```
### Here, 34.120.45.67 (under EXTERNAL-IP) is the IP address of your Load Balancer.

### 2. Point your domain to the Load Balancer
### What this means:
In your domain provider (like GoDaddy, Namecheap, or Cloudflare), create a DNS A record:
```
alloi.yourdomain.com  ->  34.120.45.67
```
### This tells the world:
“Whenever someone types alloi.yourdomain.com, send them to IP 34.120.45.67 (our load balancer).”

### In super simple words:
We are telling the "internet phonebook" that alloi.yourdomain.com should connect to our Kubernetes load balancer.


# Step 6: Configure values.yaml
### What is values.yaml?
values.yaml is like a settings file for installing Alloi.

It tells Helm (the installer):
Which domain name to use.
Which storage and database to connect to.
How to enable HTTPS.
We need to edit ./alloi-stack/values.yaml and fill in our own information.

## Breakdown of the settings
### 1. global:
This section defines general settings for Alloi.
```
global:
  domain: yourdomain.com
  hostname: alloi.yourdomain.com
  storage_class: "gp2"
  s3_bucket: your-s3-bucket-name
  aws_region: us-west-2
  database_host: "your-rds-endpoint.region.rds.amazonaws.com"
  database_port: "5432"
```
### domain: Your root domain (example: mycompany.com).
hostname: The full website address for Alloi (example: alloi.mycompany.com).
storage_class: The type of storage Kubernetes will use (e.g., "gp2" for AWS EBS volumes).
s3_bucket: The name of your S3 bucket where Alloi might store backups or files.
aws_region: The AWS region where your S3 bucket and services are located (e.g., us-east-1).
database_host: The host name of your PostgreSQL database (for example, an AWS RDS endpoint).
database_port: The port for PostgreSQL (usually 5432).

### 2. ingress:
This section controls how the website is accessed from outside (HTTP/HTTPS).
```
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: alloi.yourdomain.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - hosts:
        - alloi.yourdomain.com
      secretName: alloi-tls
```
### enabled: true means we want to use ingress (the "gatekeeper" installed earlier).
className: nginx: This tells Kubernetes to use the NGINX ingress controller we set up in Step 1.
annotations -> cert-manager.io/cluster-issuer: letsencrypt-prod:
This tells cert-manager to use the letsencrypt-prod issuer we created in Step 2 to generate an SSL certificate.

hosts:
host: alloi.yourdomain.com – The website address where Alloi will be available.
paths: – / means all requests go to Alloi.

tls:
hosts: List of domains for HTTPS (alloi.yourdomain.com).
secretName: alloi-tls is where the SSL certificate will be stored.

### In super simple words:
This step is like filling in a form with your website name, database, and storage settings.
It also tells Kubernetes: "Enable HTTPS using Let's Encrypt for alloi.yourdomain.com."

# Step 7: Configure Secrets

### What is happening in this step?
Alloi needs passwords, secret keys, and API keys to work.
These are sensitive (private) pieces of information, so we store them in a special file:
./alloi-stack/values-secrets.yaml.
This step is about filling in that file with your own credentials.

### Breakdown of values-secrets.yaml
### 1. Backend secrets
```
global:
  backend:
    secretVariables:
      DATABASE_NAME: "alloi-backend"
      DATABASE_USER: "backend_user"
      DATABASE_PASSWORD: "secure_password_4"
      DJANGO_SECRET_KEY: "your-50-character-secret-key"
      CRYPTOGRAPHY_KEY: "your-32-byte-encryption-key"
      OPENAI_API_KEY: "sk-your-openai-api-key"
      AWS_ACCESS_KEY_ID: "your-aws-access-key"
      AWS_SECRET_ACCESS_KEY: "your-aws-secret-key"
      REDIS_PASSWORD: "redis-secure-password"
      RABBITMQ_PASSWORD: "rabbitmq-secure-password"
```
### DATABASE_NAME, USER, PASSWORD: These must match the PostgreSQL database and user you created in Step 3.
DJANGO_SECRET_KEY: A long random key (50 characters). Think of it as a password for the backend application.
CRYPTOGRAPHY_KEY: A 32-byte encryption key for securely storing sensitive data.
OPENAI_API_KEY: Your OpenAI API key (if Alloi uses AI features).
AWS_ACCESS_KEY_ID & AWS_SECRET_ACCESS_KEY: Credentials to access your AWS S3 bucket.
REDIS_PASSWORD & RABBITMQ_PASSWORD: Passwords for internal services (Redis and RabbitMQ).

### 2. Embeddings secrets
```
embeddings:
  secretVariables:
    DATABASE_NAME: "alloi-embeddings"
    DATABASE_USER: "embeddings_user"
    DATABASE_PASSWORD: "secure_password_1"
    OPENAI_API_KEY: "sk-your-openai-api-key"
```
### These settings are specific to the embeddings database and also require the OpenAI key.

### 3. Supertokens database
```
supertokens:
  database:
    name: "alloi-supertokens"
    host: "your-rds-endpoint.region.rds.amazonaws.com"
    port: 5432
    user: "supertokens_user"
    password: "secure_password_3"
```
### This section tells Alloi how to connect to the supertokens database for login/authentication.

### 4. RabbitMQ and Redis
```
rabbitmq:
  auth:
    username: "alloi-rabbitmq"
    password: "rabbitmq-secure-password"
    erlangCookie: "your-erlang-cookie"

redis:
  global:
    redis:
      password: "redis-secure-password"
```
### These are the credentials for internal messaging (RabbitMQ) and caching (Redis).

### How to generate secure keys?
You can use Python commands to create strong random keys:
```
# Generate Django secret key (50 characters)
python3 -c "import secrets; print(secrets.token_urlsafe(50))"

# Generate Cryptography encryption key (32 bytes)
python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```
### Copy the output of these commands and paste them into your values-secrets.yaml.

### In simple words:
We are filling a secret locker file (values-secrets.yaml) with all passwords, API keys, and private keys that Alloi needs.
Without these, Alloi won’t be able to talk to the database, AWS, or internal services.

# Step 8: Deploy Alloi
### What are we doing in this step?
We are finally installing Alloi into Kubernetes using Helm.
We will monitor the pods (services) to check if everything is running correctly.

### 1. Deploy Alloi using Helm
```
helm install alloi ./alloi-stack -n alloi \
  -f ./alloi-stack/values.yaml \
  -f ./alloi-stack/values-secrets.yaml
```
### helm install alloi ./alloi-stack
This tells Helm to install Alloi using the files inside the alloi-stack folder.
alloi is the name of this release (you can think of it like a label).
-n alloi
Installs Alloi into the alloi namespace (we created this in Step 4).
-f ./alloi-stack/values.yaml
Uses the settings we configured earlier (Step 6).
-f ./alloi-stack/values-secrets.yaml
Uses the secrets we set up (Step 7).

### 2. Monitor Deployment
```
kubectl get pods -n alloi -w
```
### kubectl get pods -n alloi
Lists all pods (small containers running different parts of Alloi) in the alloi namespace.

-w
Means "watch" – the command will keep running and update you whenever the status of pods changes (from Pending to Running).

### What should you see?
You will see a list of pods like:
```
NAME                              READY   STATUS    RESTARTS   AGE
alloi-backend-12345               1/1     Running   0          2m
alloi-frontend-67890              1/1     Running   0          2m
```
### Wait until all the STATUS fields show Running.

### In simple words:
This is like pressing the "Install" button on an app. We tell Helm to start Alloi using our settings, and then we wait until all its pieces are up and running.

# Step 9: Verify Installation
### We are checking if Alloi is installed correctly and working.
### 1. Check all pods are running
```
kubectl get pods -n alloi
```
### This will show a list of all Alloi pods (services).
You should see all pods with STATUS as Running.
Example:
```
NAME                              READY   STATUS    RESTARTS   AGE
alloi-backend-12345               1/1     Running   0          5m
alloi-frontend-67890              1/1     Running   0          5m
```
### 2. Check SSL certificate
```
kubectl get certificates -n alloi
```
### This shows if cert-manager successfully created an SSL certificate for your domain.

You should see something like:
```
NAME        READY   SECRET       AGE
alloi-tls   True    alloi-tls    3m
```
### READY: True means HTTPS is working.

### 3. Test access from the terminal
```
curl -k https://alloi.yourdomain.com
```
### curl is a tool to test if your website is responding.
The -k option tells it to ignore SSL verification issues (just for testing).
You should get some HTML output from the Alloi website.

### Step 10: Access Alloi
### Now we use a browser to open Alloi and set it up.
Open your browser and go to:
https://alloi.yourdomain.com
You will see the Alloi setup wizard.
Create your admin account (username, email, password).
Start configuring your monitoring targets (services you want Alloi to monitor).

### In super simple words:
Step 9 is like checking if your new car’s engine and lights are working.
Step 10 is like sitting in the car for the first time, creating your driver account, and starting the ride.































