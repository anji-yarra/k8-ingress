# AWS EKS ALB Ingress Controller – Host-Based Routing

This project demonstrates how to expose applications running on **Amazon EKS** to the internet using:

- Amazon EKS
- Kubernetes Ingress
- AWS Load Balancer Controller
- AWS Application Load Balancer (ALB)
- IAM Roles for Service Accounts (IRSA)
- EKS OIDC provider
- Kubernetes Services
- Route 53
- Host-based routing

The final setup routes different DNS hostnames to different Kubernetes applications through an AWS ALB.

---

## Architecture

```text
                         Internet
                            |
                            v
                     Amazon Route 53
                            |
              +-------------+-------------+
              |                           |
              v                           v
      app1.anjidevops.online      app2.anjidevops.online
              |                           |
              +-------------+-------------+
                            |
                            v
                 AWS Application Load
                     Balancer (ALB)
                            |
                            v
                Kubernetes Ingress
                     /          \
                    /            \
                   v              v
              app-1 Service    app2-svc
                   |              |
                   v              v
               APP 1 Pods      APP 2 Pods
                   |              |
                   +------+-------+
                          |
                       EKS Nodes
```

### Request flow

```text
Browser
   |
   v
Route 53
   |
   v
ALB
   |
   v
AWS Load Balancer Controller
   |
   v
Kubernetes Ingress
   |
   +---- Host: app1.anjidevops.online ---> app-1 ---> APP 1 Pods
   |
   +---- Host: app2.anjidevops.online ---> app2-svc --> APP 2 Pods
```

---

# 1. Environment

The commands in this README were used with the following environment:

```text
EKS Cluster : roboshop-dev
AWS Region  : us-east-1
Domain      : anjidevops.online
APP 1       : app1.anjidevops.online
APP 2       : app2.anjidevops.online
```

---

# 2. Repository

Project repository:

```text
https://github.com/anji-yarra/ingress.git
```

Clone the repository:

```bash
git clone https://github.com/anji-yarra/ingress.git
cd ingress
```

---

# 3. Check the EKS Cluster

Verify the cluster:

```bash
eksctl get cluster --region us-east-1
```

Verify kubectl access:

```bash
kubectl get nodes
```

Example:

```text
NAME                          STATUS   ROLES    AGE
ip-10-0-11-184.ec2.internal   Ready    <none>   ...
ip-10-0-12-207.ec2.internal   Ready    <none>   ...
```

---

# 4. Verify the EKS OIDC Provider

AWS Load Balancer Controller uses **IRSA (IAM Roles for Service Accounts)**.

First retrieve the OIDC issuer:

```bash
aws eks describe-cluster \
  --name roboshop-dev \
  --region us-east-1 \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

Expected format:

```text
https://oidc.eks.us-east-1.amazonaws.com/id/<OIDC_ID>
```

Associate the OIDC provider:

```bash
eksctl utils associate-iam-oidc-provider \
  --region us-east-1 \
  --cluster roboshop-dev \
  --approve
```

If it is already configured, eksctl reports:

```text
IAM Open ID Connect provider is already associated with cluster
```

---

# 5. Download AWS Load Balancer Controller IAM Policy

Download the IAM policy:

```bash
curl -o iam-policy.json \
https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v3.4.2/docs/install/iam_policy.json
```

Verify:

```bash
ls -lh iam-policy.json
```

Check the beginning of the file:

```bash
head -5 iam-policy.json
```

---

# 6. Create the IAM Policy

Create the policy:

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json
```

If AWS returns:

```text
EntityAlreadyExists
```

the policy already exists.

Find the correct AWS account ID:

```bash
aws sts get-caller-identity
```

Example:

```json
{
    "UserId": "...",
    "Account": "884057990406",
    "Arn": "arn:aws:sts::884057990406:assumed-role/roboshop-dev-bastion/..."
}
```

List the policy:

```bash
aws iam list-policies \
  --scope Local \
  --query "Policies[?PolicyName=='AWSLoadBalancerControllerIAMPolicy'].[PolicyName,Arn,DefaultVersionId]" \
  --output table
```

The policy ARN used in this setup was:

```text
arn:aws:iam::884057990406:policy/AWSLoadBalancerControllerIAMPolicy
```

> Important: AWS account IDs are environment-specific. Do not blindly copy the account ID into another AWS account.

---

# 7. Create the IAM Service Account / IRSA

Create the IAM service account:

```bash
eksctl create iamserviceaccount \
  --cluster=roboshop-dev \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::884057990406:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region=us-east-1 \
  --approve
```

Verify it from eksctl:

```bash
eksctl get iamserviceaccount \
  --cluster=roboshop-dev \
  --region=us-east-1
```

Expected:

```text
NAMESPACE       NAME                            ROLE ARN
kube-system     aws-load-balancer-controller    arn:aws:iam::884057990406:role/eksctl-roboshop-dev-addon-iamserviceaccount-...
```

---

# 8. Troubleshooting the Service Account

During this setup, `eksctl get iamserviceaccount` showed the IAM role, but Kubernetes initially did not show the ServiceAccount:

```bash
kubectl get sa -n kube-system | grep load-balancer
```

The IAM service account existed in eksctl, but the Kubernetes ServiceAccount was missing.

We created it manually:

```bash
kubectl create serviceaccount aws-load-balancer-controller \
  -n kube-system
```

Then we annotated it with the IAM role:

```bash
kubectl annotate serviceaccount aws-load-balancer-controller \
  -n kube-system \
  eks.amazonaws.com/role-arn=arn:aws:iam::884057990406:role/eksctl-roboshop-dev-addon-iamserviceaccount-k-Role1-xQbXadi2uT7P
```

Verify:

```bash
kubectl get sa aws-load-balancer-controller \
  -n kube-system \
  -o yaml
```

Expected annotation:

```yaml
annotations:
  eks.amazonaws.com/role-arn: arn:aws:iam::884057990406:role/eksctl-roboshop-dev-addon-iamserviceaccount-k-Role1-xQbXadi2uT7P
```

---

# 9. Verify the IAM Trust Policy

Check the role:

```bash
aws iam get-role \
  --role-name eksctl-roboshop-dev-addon-iamserviceaccount-k-Role1-xQbXadi2uT7P \
  --query 'Role.AssumeRolePolicyDocument' \
  --output json
```

The trust relationship must reference the **same OIDC provider ID returned by the EKS cluster**.

It should contain:

```json
{
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Principal": {
    "Federated": "arn:aws:iam::884057990406:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/<OIDC_ID>"
  },
  "Condition": {
    "StringEquals": {
      "oidc.eks.us-east-1.amazonaws.com/id/<OIDC_ID>:aud": "sts.amazonaws.com",
      "oidc.eks.us-east-1.amazonaws.com/id/<OIDC_ID>:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller"
    }
  }
}
```

This is critical for IRSA.

---

# 10. Install AWS Load Balancer Controller

After the ServiceAccount and IAM role are available, deploy the AWS Load Balancer Controller.

The controller must use:

```text
ServiceAccount:
kube-system/aws-load-balancer-controller
```

Verify the deployment:

```bash
kubectl get deployment -n kube-system
```

Expected:

```text
NAME                           READY   UP-TO-DATE   AVAILABLE
aws-load-balancer-controller   2/2     2            2
coredns                        2/2     2            2
metrics-server                 2/2     2            2
```

Verify the pods:

```bash
kubectl get pods -n kube-system | grep aws-load-balancer
```

Expected:

```text
aws-load-balancer-controller-xxxxx   1/1   Running
aws-load-balancer-controller-xxxxx   1/1   Running
```

---

# 11. Verify IngressClass

Check:

```bash
kubectl get ingressclass
```

Expected:

```text
NAME   CONTROLLER
alb    ingress.k8s.aws/alb
```

This means Kubernetes has an IngressClass called `alb`, managed by AWS Load Balancer Controller.

---

# 12. Application 1

APP 1 uses nginx.

Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app-1
  template:
    metadata:
      labels:
        app: app-1
    spec:
      containers:
      - name: app1
        image: nginx
        ports:
        - containerPort: 80
```

Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-1
spec:
  selector:
    app: app-1
  ports:
  - port: 80
    targetPort: 80
```

Apply:

```bash
kubectl apply -f app1.yaml
```

---

# 13. APP 1 Ingress

The APP 1 Ingress uses:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-1
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
  - host: app1.anjidevops.online
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-1
            port:
              number: 80
```

Important annotations:

### Internet-facing ALB

```yaml
alb.ingress.kubernetes.io/scheme: internet-facing
```

This creates an ALB reachable from the internet.

### IP target mode

```yaml
alb.ingress.kubernetes.io/target-type: ip
```

The ALB sends traffic directly to Pod IP addresses.

Example target IPs from this environment:

```text
10.0.11.4:80
10.0.12.147:80
```

---

# 14. APP 2

APP 2 uses a custom HTML page to make host-based routing easy to identify.

Example page:

```html
<html>
<body style="background-color: lightgreen;">
  <h1>🎯 APP 2</h1>
  <h2>app2.anjidevops.online</h2>
  <p>This request reached APP 2</p>
</body>
</html>
```

APP 2 uses:

```text
Deployment: app2
Service:     app2-svc
Labels:      app=app2
```

Verify:

```bash
kubectl get pods --show-labels
```

Example:

```text
app2-6dfbdf78-g5m4w    1/1   Running   app=app2
app2-6dfbdf78-wnmdp    1/1   Running   app=app2
```

Service:

```text
app2-svc
```

---

# 15. Host-Based Ingress

The second Ingress routes two hostnames through one ALB:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: apps
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb

  rules:
  - host: app1.anjidevops.online
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-1
            port:
              number: 80

  - host: app2.anjidevops.online
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-svc
            port:
              number: 80
```

Apply:

```bash
kubectl apply -f apps.yaml
```

Check:

```bash
kubectl get ingress
```

Example:

```text
NAME    CLASS   HOSTS                                           ADDRESS
app-1   alb     app1.anjidevops.online                          k8s-default-app1-....elb.amazonaws.com
apps    alb     app1.anjidevops.online,app2.anjidevops.online   k8s-default-apps-....elb.amazonaws.com
```

---

# 16. Verify Ingress

```bash
kubectl describe ingress apps
```

Check the rules and backend services.

You should see:

```text
app1.anjidevops.online --> app-1:80
app2.anjidevops.online --> app2-svc:80
```

---

# 17. Verify ALB

Get the Ingress address:

```bash
kubectl get ingress
```

Example:

```text
k8s-default-apps-7cb23d8e54-1126853669.us-east-1.elb.amazonaws.com
```

The AWS Load Balancer Controller creates the ALB automatically from the Kubernetes Ingress.

---

# 18. Test the ALB Directly

Because ALB host-based routing uses the HTTP `Host` header, we can test the ALB directly.

### APP 2

```bash
curl -H "Host: app2.anjidevops.online" \
http://k8s-default-apps-7cb23d8e54-1126853669.us-east-1.elb.amazonaws.com
```

Expected:

```html
<html>
<body style="background-color: lightgreen;">
  <h1>🎯 APP 2</h1>
  <h2>app2.anjidevops.online</h2>
  <p>This request reached APP 2</p>
</body>
</html>
```

This test successfully returned:

```text
HTTP/1.1 200 OK
Server: nginx/1.31.3
```

---

# 19. Test from Windows

From a Windows machine:

```cmd
curl.exe -v -H "Host: app2.anjidevops.online" http://<ALB-DNS>
```

The test confirmed:

```text
HTTP/1.1 200 OK
```

and returned the APP 2 custom HTML.

This proves:

```text
Windows Internet
       |
       v
AWS ALB
       |
       v
Ingress host rule
       |
       v
app2-svc
       |
       v
APP 2 Pod
```

---

# 20. Route 53

Create a Route 53 record for:

```text
app1.anjidevops.online
```

and:

```text
app2.anjidevops.online
```

Recommended configuration:

```text
Record type: A
Alias: Yes
Target: Application Load Balancer
Region: US East (N. Virginia)
```

Do not manually enter ALB IP addresses.

The Alias record should point to the ALB DNS name.

---

# 21. Verify DNS

From Linux:

```bash
dig app1.anjidevops.online
```

Example:

```text
ANSWER SECTION:
app1.anjidevops.online. 60 IN A 52.86.148.153
app1.anjidevops.online. 60 IN A 54.163.70.21
```

From Windows:

```cmd
nslookup app2.anjidevops.online
```

---

# 22. End-to-End Test

From the internet:

```cmd
curl.exe -v http://app1.anjidevops.online
```

and:

```cmd
curl.exe -v http://app2.anjidevops.online
```

APP 2 should return:

```text
🎯 APP 2
app2.anjidevops.online
This request reached APP 2
```

---

# 23. Debugging Commands

### Kubernetes resources

```bash
kubectl get pods
kubectl get svc
kubectl get ingress
kubectl get ingressclass
```

### Detailed Ingress information

```bash
kubectl describe ingress apps
```

### Controller

```bash
kubectl get deployment -n kube-system
kubectl get pods -n kube-system | grep aws-load-balancer
```

### Events

```bash
kubectl get events --sort-by=.lastTimestamp
```

### Service endpoints

```bash
kubectl get endpoints
```

or:

```bash
kubectl get endpointslices
```

---

# 24. AWS ALB Debugging

Find the ALB:

```bash
aws elbv2 describe-load-balancers \
  --region us-east-1
```

Get the ALB ARN:

```bash
ALB_ARN=$(aws elbv2 describe-load-balancers \
  --region us-east-1 \
  --query "LoadBalancers[?DNSName=='<ALB-DNS>'].LoadBalancerArn" \
  --output text)

echo $ALB_ARN
```

Check listeners:

```bash
aws elbv2 describe-listeners \
  --load-balancer-arn "$ALB_ARN" \
  --region us-east-1
```

Check listener rules:

```bash
aws elbv2 describe-rules \
  --listener-arn $(aws elbv2 describe-listeners \
  --load-balancer-arn "$ALB_ARN" \
  --region us-east-1 \
  --query 'Listeners[0].ListenerArn' \
  --output text) \
  --region us-east-1
```

Check target groups:

```bash
aws elbv2 describe-target-groups \
  --load-balancer-arn "$ALB_ARN" \
  --region us-east-1
```

Check target health:

```bash
TG_ARN=$(aws elbv2 describe-target-groups \
  --load-balancer-arn "$ALB_ARN" \
  --region us-east-1 \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text)

aws elbv2 describe-target-health \
  --target-group-arn "$TG_ARN" \
  --region us-east-1
```

Expected:

```text
TargetHealth:
    State: healthy
```

---

# 25. Important Error We Encountered

## Error: Deployment version `app/v1`

Initially the Deployment contained:

```yaml
apiVersion: app/v1
```

This produced:

```text
no matches for kind "Deployment" in version "app/v1"
```

The correct API version is:

```yaml
apiVersion: apps/v1
```

---

# 26. YAML Typo

Another error was:

```yaml
spec:
  template:
    meatadata:
```

The correct field is:

```yaml
spec:
  template:
    metadata:
```

A dry run helped validate the YAML:

```bash
kubectl apply --dry-run=client -f app1.yaml
```

Expected:

```text
deployment.apps/app1 created (dry run)
service/app-1 configured (dry run)
ingress.networking.k8s.io/app-1 configured (dry run)
```

---

# 27. IRSA Error

We encountered:

```text
AccessDenied: Not authorized to perform sts:AssumeRoleWithWebIdentity
```

This means the AWS Load Balancer Controller could not assume its IAM role through IRSA.

The important things to verify are:

```text
1. EKS OIDC provider exists
2. IAM role trust policy uses the correct OIDC provider
3. ServiceAccount has the correct role ARN annotation
4. ServiceAccount namespace is kube-system
5. ServiceAccount name is aws-load-balancer-controller
6. Trust policy contains sts:AssumeRoleWithWebIdentity
```

Check the ServiceAccount:

```bash
kubectl get sa aws-load-balancer-controller \
  -n kube-system \
  -o yaml
```

Check the IAM trust policy:

```bash
aws iam get-role \
  --role-name <ROLE_NAME> \
  --query 'Role.AssumeRolePolicyDocument' \
  --output json
```

---

# 28. Webhook Error

We also encountered:

```text
failed calling webhook "vingress.elbv2.k8s.aws"
```

with:

```text
no endpoints available for service
"aws-load-balancer-webhook-service"
```

This happened while the AWS Load Balancer Controller/webhook was not yet ready.

Check:

```bash
kubectl get pods -n kube-system | grep aws-load-balancer
```

and:

```bash
kubectl get svc -n kube-system | grep aws-load-balancer
```

The controller should eventually become:

```text
2/2 Ready
```

---

# 29. Backend Service Does Not Exist

We also encountered:

```text
Backend service does not exist
```

This happened when an Ingress rule referenced a Service name that did not exist.

Current Services:

```text
app-1
app2-svc
```

Therefore the Ingress must reference:

```yaml
backend:
  service:
    name: app-1
```

or:

```yaml
backend:
  service:
    name: app2-svc
```

Check:

```bash
kubectl get svc
```

and compare the Service names with the Ingress configuration.

---

# 30. Why We Use Ingress

A Kubernetes Service provides access to an application inside the cluster.

An Ingress provides HTTP/HTTPS routing rules.

For example:

```text
app1.anjidevops.online
        |
        v
      Ingress
        |
        v
     app-1 Service
```

and:

```text
app2.anjidevops.online
        |
        v
      Ingress
        |
        v
     app2-svc
```

The AWS Load Balancer Controller watches the Ingress and creates/configures an AWS ALB.

---

# 31. Why AWS Load Balancer Controller?

Kubernetes itself does not automatically create an AWS ALB just because an Ingress object exists.

The AWS Load Balancer Controller watches Kubernetes resources and translates them into AWS resources.

```text
Kubernetes Ingress
        |
        v
AWS Load Balancer Controller
        |
        +----> AWS ALB
        |
        +----> Listener
        |
        +----> Listener Rules
        |
        +----> Target Groups
```

---

# 32. Why OIDC + IRSA?

The AWS Load Balancer Controller needs permission to create and manage AWS resources.

Instead of giving AWS permissions to every EC2 node, we use:

```text
EKS OIDC
    |
    v
IAM Role
    |
    v
Kubernetes ServiceAccount
    |
    v
AWS Load Balancer Controller
```

This is called:

**IRSA – IAM Roles for Service Accounts**

The controller gets only the IAM permissions it needs.

---

# 33. Target Type: IP

We used:

```yaml
alb.ingress.kubernetes.io/target-type: ip
```

Therefore the ALB target group contains Pod IPs.

Example:

```text
ALB
 |
 +--> 10.0.11.4:80
 |
 +--> 10.0.12.147:80
```

Both targets were verified as:

```text
healthy
```

This means the ALB can directly send traffic to the application Pods.

---

# 34. Host-Based Routing

The most important concept demonstrated in this project is host-based routing.

The ALB receives:

```http
Host: app1.anjidevops.online
```

and routes to:

```text
app-1
```

When it receives:

```http
Host: app2.anjidevops.online
```

it routes to:

```text
app2-svc
```

Example:

```text
                         ALB
                          |
              +-----------+-----------+
              |                       |
        Host = app1              Host = app2
              |                       |
              v                       v
          app-1 Service          app2-svc
              |                       |
              v                       v
          APP 1 Pods              APP 2 Pods
```

---

# 35. Final Verification

Check all components:

```bash
kubectl get nodes
kubectl get pods
kubectl get svc
kubectl get ingressclass
kubectl get ingress
kubectl get pods -n kube-system | grep aws-load-balancer
```

Check AWS:

```bash
aws sts get-caller-identity
aws eks describe-cluster --name roboshop-dev --region us-east-1
aws elbv2 describe-load-balancers --region us-east-1
```

Check DNS:

```bash
dig app1.anjidevops.online
dig app2.anjidevops.online
```

Test applications:

```bash
curl http://app1.anjidevops.online
curl http://app2.anjidevops.online
```

---

# 36. Final Architecture Summary

```text
                    Internet
                       |
                       v
                  Route 53
                       |
          +------------+------------+
          |                         |
          v                         v
 app1.anjidevops.online     app2.anjidevops.online
          |                         |
          +------------+------------+
                       |
                       v
                 AWS ALB
                       |
                       v
            Kubernetes Ingress
                       |
            AWS Load Balancer
                 Controller
                       |
             +---------+---------+
             |                   |
             v                   v
         app-1 Service       app2-svc
             |                   |
             v                   v
          APP 1 Pods          APP 2 Pods
             |                   |
             +---------+---------+
                       |
                    EKS Nodes
```

## Technologies

- AWS EKS
- Kubernetes
- AWS Load Balancer Controller
- Application Load Balancer
- IAM
- OIDC
- IRSA
- Route 53
- nginx
- kubectl
- eksctl
- AWS CLI

---

## Key Takeaways

1. **Ingress** defines HTTP/HTTPS routing rules.
2. **AWS Load Balancer Controller** watches the Ingress and creates/configures the AWS ALB.
3. **OIDC** establishes trust between EKS and AWS IAM.
4. **IRSA** allows the Kubernetes ServiceAccount to assume an IAM role.
5. The controller uses that IAM role to manage AWS load-balancing resources.
6. `target-type: ip` sends ALB traffic directly to Pod IPs.
7. **Route 53** maps application DNS names to the ALB.
8. Host-based routing allows multiple applications to be exposed through ALB rules.
9. Kubernetes Services provide the backend abstraction for the applications.
10. ALB target health must be `healthy` for traffic to reach the Pods.

---

## Cleanup

Before deleting resources, remove the Kubernetes Ingresses:

```bash
kubectl delete ingress app-1
kubectl delete ingress apps
```

Then remove applications:

```bash
kubectl delete -f app1.yaml
```

Remove APP 2 resources according to the manifests used in the project.

> Verify the AWS ALBs and target groups are removed before considering the cleanup complete.
