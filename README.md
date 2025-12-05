# CKAD Exam Questions - Practice Set 2

## Question 1: Create and Use Secrets

**Scenario:** A deployment has three hardcoded environment variables: `postgres_user`, `postgres_database`, and `postgres_password`.

**Task:** 
1. Create a Secret named `postgres` with keys: `username`, `database`, `password`
2. Update the deployment to use this Secret instead of hardcoded values

**Solution:**
```bash
# Create the secret
kubectl create secret generic postgres \
  --from-literal=username=postgres \
  --from-literal=database=mydb \
  --from-literal=password=secretpassword

# Or create from file
kubectl create secret generic postgres \
  --from-literal=username=<value> \
  --from-literal=database=<value> \
  --from-literal=password=<value>

# Update deployment to use secret
kubectl set env deployment/<deployment-name> \
  --from=secret/postgres \
  --prefix=POSTGRES_

# Or edit deployment and update env section:
# env:
#   - name: POSTGRES_USER
#     valueFrom:
#       secretKeyRef:
#         name: postgres
#         key: username
#   - name: POSTGRES_DATABASE
#     valueFrom:
#       secretKeyRef:
#         name: postgres
#         key: database
#   - name: POSTGRES_PASSWORD
#     valueFrom:
#       secretKeyRef:
#         name: postgres
#         key: password
```

---

## Question 2: Resource Requests and Limits with Resource Quota

**Scenario:** Update a deployment to use resource requests (100m CPU, 128Mi memory) and limits (double the requests). Ensure the pod runs successfully.

**Note:** A ResourceQuota exists that prevents the deployment from terminating and scaling. Fix this issue.

**Solution:**
```bash
# First, check the resource quota
kubectl get resourcequota -n <namespace>
kubectl describe resourcequota <quota-name> -n <namespace>

# Update deployment with resources
kubectl set resources deployment/<deployment-name> \
  --requests=cpu=100m,memory=128Mi \
  --limits=cpu=200m,memory=256Mi

# If quota is blocking, you may need to:
# 1. Delete old pods first to free up quota
kubectl delete pod <old-pod-name> -n <namespace>

# 2. Or scale down first, then update, then scale up
kubectl scale deployment/<deployment-name> --replicas=0
kubectl set resources deployment/<deployment-name> \
  --requests=cpu=100m,memory=128Mi \
  --limits=cpu=200m,memory=256Mi
kubectl scale deployment/<deployment-name> --replicas=<desired-replicas>

# Verify pod is running
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
```

---

## Question 3: Update Container Name and Image Tag (Paused Deployment)

**Scenario:** Change the container name in a deployment to `musli` and update the image tag to `musli`. Ensure the new pod runs successfully.

**Note:** The deployment is paused. Use `kubectl rollout resume deployment <name>`.

**Solution:**
```bash
# Update container name and image
kubectl set image deployment/<deployment-name> \
  <old-container-name>=<image>:musli

# Or edit the deployment
kubectl edit deployment/<deployment-name>
# Change:
# - name: <old-name>  ->  - name: musli
# image: <image>:<tag>  ->  image: <image>:musli

# Resume the paused deployment
kubectl rollout resume deployment/<deployment-name>

# Verify rollout
kubectl rollout status deployment/<deployment-name>

# Check new pod is running
kubectl get pods -n <namespace>
```

---

## Question 4: Fix RBAC Issue - RoleBinding

**Scenario:** Pods in a deployment are failing to run. Check the logs and identify the issue.

**Note:** The pods are trying to list pods in the namespace using the default ServiceAccount, which doesn't have access. A ServiceAccount and Role have been created. You need to create a RoleBinding and use it in the deployment.

**Solution:**
```bash
# Check pod logs
kubectl logs <pod-name> -n <namespace>

# Check existing ServiceAccount and Role
kubectl get sa -n <namespace>
kubectl get role -n <namespace>
kubectl describe role <role-name> -n <namespace>

# Create RoleBinding
kubectl create rolebinding <binding-name> \
  --role=<role-name> \
  --serviceaccount=<namespace>:<serviceaccount-name> \
  -n <namespace>

# Or create from YAML
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: <binding-name>
  namespace: <namespace>
subjects:
- kind: ServiceAccount
  name: <serviceaccount-name>
  namespace: <namespace>
roleRef:
  kind: Role
  name: <role-name>
  apiGroup: rbac.authorization.k8s.io/v1
EOF

# Update deployment to use the ServiceAccount
kubectl set serviceaccount deployment/<deployment-name> <serviceaccount-name> -n <namespace>

# Or edit deployment
kubectl edit deployment/<deployment-name>
# Add:
# spec:
#   template:
#     spec:
#       serviceAccountName: <serviceaccount-name>

# Verify pod runs successfully
kubectl get pods -n <namespace>
kubectl logs <pod-name> -n <namespace>
```

---

## Question 5: NetworkPolicy - Allow Traffic Between Pods

**Scenario:** You have a pod called `newpod` in the namespace. Make ingress and egress traffic pass between this pod and the `db` and `frontend` pods.

**Note:** The pods are created and a NetworkPolicy exists, but the selector labels `db-access=true` and `frontend-access=true` are missing on `newpod`.

**Solution:**
```bash
# Check existing NetworkPolicy
kubectl get networkpolicy -n <namespace>
kubectl describe networkpolicy <policy-name> -n <namespace>

# Add required labels to newpod
kubectl label pod newpod db-access=true frontend-access=true -n <namespace>

# Or patch the pod
kubectl patch pod newpod -n <namespace> -p '{"metadata":{"labels":{"db-access":"true","frontend-access":"true"}}}'

# Verify labels
kubectl get pod newpod -n <namespace> --show-labels

# Test connectivity
kubectl exec newpod -n <namespace> -- ping <db-pod-ip>
kubectl exec newpod -n <namespace> -- ping <frontend-pod-ip>
```

---

## Question 6: Canary Deployment with Traffic Splitting

**Scenario:** You have a deployment called `current-web-deployment` with 5 replicas and a service that routes to it. Create a new deployment called `canary` with image `nginx:stable` and route 20% of traffic to the new pods while keeping the maximum pods in the namespace to 10.

**Solution:**
```bash
# Check current deployment
kubectl get deployment current-web-deployment -n <namespace>
kubectl get svc -n <namespace>

# Create canary deployment (2 replicas for 20% of 10 max pods)
kubectl create deployment canary \
  --image=nginx:stable \
  --replicas=2 \
  -n <namespace>

# Add labels to match service selector
kubectl label deployment canary app=web -n <namespace>

# Update service selector if needed, or use label selector
# The service should select pods with the same label
# Traffic will be distributed by kube-proxy (round-robin)

# Alternative: Use service with multiple selectors or create separate service
# and use Ingress with traffic splitting

# Verify total pods don't exceed 10
kubectl get pods -n <namespace> --no-headers | wc -l

# Scale if needed
kubectl scale deployment current-web-deployment --replicas=8 -n <namespace>
kubectl scale deployment canary --replicas=2 -n <namespace>
```

---

## Question 7: Create Ingress Resource

**Scenario:** You have a deployment called `web-app` with a service called `web-svc`. Create an Ingress resource to make it accessible at `web-app.local` with path `/`.

**Solution:**
```bash
# Check deployment and service
kubectl get deployment web-app -n <namespace>
kubectl get svc web-svc -n <namespace>
kubectl describe svc web-svc -n <namespace>

# Create Ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  namespace: <namespace>
spec:
  rules:
  - host: web-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 80
EOF

# Verify ingress
kubectl get ingress -n <namespace>
kubectl describe ingress web-app-ingress -n <namespace>

# Test (add to /etc/hosts if needed)
curl -H "Host: web-app.local" http://<ingress-ip>/
```

---

## Question 8: Fix Ingress Configuration

**Scenario:** Fix the Ingress resource to make it accessible at `content-marlin.local/content-marlin`. All files are available at `/home/candidate/content-marlin`.

**Note:** You will find deployment, service, and ingress YAML files. The Ingress doesn't have the host configured, and the service is misconfigured (wrong port).

**Solution:**
```bash
# Check existing resources
kubectl get deployment,svc,ingress -n <namespace>
kubectl get deployment -o yaml -n <namespace> > /home/candidate/content-marlin/deployment.yaml
kubectl get svc -o yaml -n <namespace> > /home/candidate/content-marlin/service.yaml
kubectl get ingress -o yaml -n <namespace> > /home/candidate/content-marlin/ingress.yaml

# Check service port
kubectl describe svc <service-name> -n <namespace>

# Fix service port (if misconfigured)
kubectl edit svc <service-name> -n <namespace>
# Update port to match container port (usually 80 or 8080)

# Fix ingress
kubectl edit ingress <ingress-name> -n <namespace>
# Add host and fix path:
# spec:
#   rules:
#   - host: content-marlin.local
#     http:
#       paths:
#       - path: /content-marlin
#         pathType: Prefix
#         backend:
#           service:
#             name: <service-name>
#             port:
#               number: <correct-port>

# Or apply fixed YAML
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <ingress-name>
  namespace: <namespace>
spec:
  rules:
  - host: content-marlin.local
    http:
      paths:
      - path: /content-marlin
        pathType: Prefix
        backend:
          service:
            name: <service-name>
            port:
              number: <correct-port>
EOF

# Verify
kubectl get ingress -n <namespace>
curl -H "Host: content-marlin.local" http://<ingress-ip>/content-marlin
```

---

## Question 9: Fix RBAC Issue - Use Existing ServiceAccount

**Scenario:** A deployment has an error. The pod is trying to list pods in the namespace while using the default ServiceAccount, which doesn't have the necessary permissions.

**Note:** The Role and RoleBinding are already created, and a ServiceAccount has been created. Find the ServiceAccount and use it in the deployment.

**Solution:**
```bash
# Check pod logs to confirm the error
kubectl logs <pod-name> -n <namespace>
kubectl describe pod <pod-name> -n <namespace>

# Find existing ServiceAccounts in the namespace
kubectl get sa -n <namespace>

# Check which ServiceAccount has the RoleBinding
kubectl get rolebinding -n <namespace>
kubectl describe rolebinding <rolebinding-name> -n <namespace>

# The output will show which ServiceAccount is bound to the role
# Look for the ServiceAccount name in the subjects section

# Alternatively, check all ServiceAccounts and their associated RoleBindings
for sa in $(kubectl get sa -n <namespace> -o name); do
  echo "Checking $sa:"
  kubectl get rolebinding -n <namespace> -o jsonpath='{.items[*].subjects[?(@.kind=="ServiceAccount")].name}' | grep -q $(echo $sa | cut -d/ -f2) && echo "  Has RoleBinding"
done

# Once you identify the ServiceAccount, update the deployment
kubectl set serviceaccount deployment/<deployment-name> <serviceaccount-name> -n <namespace>

# Or edit the deployment
kubectl edit deployment/<deployment-name> -n <namespace>
# Update:
# spec:
#   template:
#     spec:
#       serviceAccountName: <serviceaccount-name>

# Verify the fix
kubectl get pods -n <namespace>
kubectl describe pod <new-pod-name> -n <namespace>
# Check that serviceAccountName is set correctly

# Test that the pod can now list pods
kubectl exec <pod-name> -n <namespace> -- kubectl get pods -n <namespace>
``` 

---

## Question 10: Security Context - RunAsUser and Capabilities

**Scenario:** Update the deployment to run with user ID 10000 and add the capability `NET_BIND_SERVICE`.

**Solution:**
```bash
# Update deployment security context
kubectl patch deployment <deployment-name> -n <namespace> --type='json' \
  -p='[
    {
      "op": "add",
      "path": "/spec/template/spec/securityContext",
      "value": {
        "runAsUser": 10000
      }
    },
    {
      "op": "add",
      "path": "/spec/template/spec/containers/0/securityContext",
      "value": {
        "capabilities": {
          "add": ["NET_BIND_SERVICE"]
        }
      }
    }
  ]'

# Or edit deployment
kubectl edit deployment <deployment-name> -n <namespace>
# Add:
# spec:
#   template:
#     spec:
#       securityContext:
#         runAsUser: 10000
#       containers:
#       - name: <container-name>
#         securityContext:
#           capabilities:
#             add:
#             - NET_BIND_SERVICE

# Verify
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
```

---

## Question 11: Build Docker Image

**Scenario:** Use any container tool (docker, podman, etc.) to build the Dockerfile at `/tmp/app/Dockerfile` with the name `grapper` and tag `12`.

**Solution:**
```bash
# Using Docker
docker build -t grapper:12 /tmp/app/Dockerfile

# Or if Dockerfile is in /tmp/app/
docker build -t grapper:12 -f /tmp/app/Dockerfile /tmp/app/

# Using Podman
podman build -t grapper:12 -f /tmp/app/Dockerfile /tmp/app/

# Verify image
docker images | grep grapper
# or
podman images | grep grapper
```

---

## Question 12: Add Readiness Probe

**Scenario:** Add a readiness probe at `/healthz` with `initialDelaySeconds` of 5 and `periodSeconds` of 3.

**Solution:**
```bash
# Update deployment with readiness probe
kubectl patch deployment <deployment-name> -n <namespace> --type='json' \
  -p='[
    {
      "op": "add",
      "path": "/spec/template/spec/containers/0/readinessProbe",
      "value": {
        "httpGet": {
          "path": "/healthz",
          "port": 8080
        },
        "initialDelaySeconds": 5,
        "periodSeconds": 3
      }
    }
  ]'

# Or edit deployment
kubectl edit deployment <deployment-name> -n <namespace>
# Add:
# containers:
# - name: <container-name>
#   readinessProbe:
#     httpGet:
#       path: /healthz
#       port: 80
#     initialDelaySeconds: 5
#     periodSeconds: 3

# Verify
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
```

---

## Question 13: Update Deployment Strategy and Container

**Scenario:** Update the deployment strategy to have `maxSurge` equal to 5% and `maxUnavailable` equal to 5%. Also change the container name and image as specified.

**Solution:**
```bash
# Update deployment strategy and container
kubectl patch deployment <deployment-name> -n <namespace> --type='json' \
  -p='[
    {
      "op": "replace",
      "path": "/spec/strategy/rollingUpdate",
      "value": {
        "maxSurge": "5%",
        "maxUnavailable": "5%"
      }
    },
    {
      "op": "replace",
      "path": "/spec/template/spec/containers/0/name",
      "value": "<new-container-name>"
    },
    {
      "op": "replace",
      "path": "/spec/template/spec/containers/0/image",
      "value": "<new-image>"
    }
  ]'

# Or edit deployment
kubectl edit deployment <deployment-name> -n <namespace>
# Update:
# spec:
#   strategy:
#     type: RollingUpdate
#     rollingUpdate:
#       maxSurge: 5%
#       maxUnavailable: 5%
#   template:
#     spec:
#       containers:
#       - name: <new-container-name>
#         image: <new-image>

# Verify rollout
kubectl rollout status deployment/<deployment-name> -n <namespace>
kubectl get pods -n <namespace>
```

---

## Quick Reference Commands

```bash
# Secrets
kubectl create secret generic <name> --from-literal=<key>=<value>
kubectl set env deployment/<name> --from=secret/<secret-name> --prefix=<PREFIX>_

# Resources
kubectl set resources deployment/<name> --requests=cpu=100m,memory=128Mi --limits=cpu=200m,memory=256Mi

# Images and containers
kubectl set image deployment/<name> <container>=<image>:<tag>
kubectl rollout resume deployment/<name>

# RBAC
kubectl create rolebinding <name> --role=<role> --serviceaccount=<ns>:<sa> -n <ns>
kubectl set serviceaccount deployment/<name> <sa-name>

# Labels
kubectl label pod <name> <key>=<value> -n <ns>

# Ingress
kubectl create ingress <name> --rule="<host>/*=<svc>:<port>"

# Security Context
kubectl patch deployment <name> -p '{"spec":{"template":{"spec":{"securityContext":{"runAsUser":10000}}}}}'

# Probes
kubectl patch deployment <name> -p '{"spec":{"template":{"spec":{"containers":[{"name":"<name>","readinessProbe":{"httpGet":{"path":"/healthz","port":80},"initialDelaySeconds":5,"periodSeconds":3}}]}}}}'

# Strategy
kubectl patch deployment <name> -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":"5%","maxUnavailable":"5%"}}}}'

# LimitRange
kubectl set resources deployment/<name> --requests=memory=200Mi --limits=memory=320Mi
```

---

## Question 14: Fix Memory Issues with LimitRange

**Scenario:** The pods of the deployment `mem-consume` cannot run because of memory issues. Set the memory request to 200Mi and the limit equal to half of the enforced limit in this namespace.

**Note:** There is a LimitRange resource created with a limit of 640Mi, so half of it is 320Mi. Add both request and limit.

**Solution:**
```bash
# Check the LimitRange to confirm the enforced limit
kubectl get limitrange -n <namespace>
kubectl describe limitrange <limitrange-name> -n <namespace>

# The LimitRange shows max limit of 640Mi, so half is 320Mi
# Set memory request to 200Mi and limit to 320Mi
kubectl set resources deployment/mem-consume \
  --requests=memory=200Mi \
  --limits=memory=320Mi \
  -n <namespace>

# Or edit the deployment
kubectl edit deployment/mem-consume -n <namespace>
# Update:
# spec:
#   template:
#     spec:
#       containers:
#       - name: <container-name>
#         resources:
#           requests:
#             memory: 200Mi
#           limits:
#             memory: 320Mi

# Verify the pods can now run
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>

# Check that resources are set correctly
kubectl get deployment mem-consume -n <namespace> -o jsonpath='{.spec.template.spec.containers[0].resources}'
```
