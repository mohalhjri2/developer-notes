# Kubernetes — The Complete Field Guide

> "Kubernetes is Greek for 'helmsman.' Someone still has to know where the ship is going."

---

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [kubectl Essentials](#kubectl-essentials)
3. [Pods](#pods)
4. [Deployments](#deployments)
5. [Services](#services)
6. [ConfigMaps & Secrets](#configmaps--secrets)
7. [Volumes & PersistentVolumes](#volumes--persistentvolumes)
8. [Namespaces](#namespaces)
9. [RBAC](#rbac)
10. [Ingress](#ingress)
11. [StatefulSets](#statefulsets)
12. [DaemonSets & Jobs](#daemonsets--jobs)
13. [Resource Management](#resource-management)
14. [Probes](#probes)
15. [Horizontal Pod Autoscaler](#horizontal-pod-autoscaler)
16. [Helm](#helm)
17. [Debugging & Troubleshooting](#debugging--troubleshooting)
18. [Production Checklist](#production-checklist)

---

## Core Concepts

```
Control Plane
├── kube-apiserver      Gateway; all kubectl commands go here
├── etcd                Cluster state database
├── kube-scheduler      Assigns pods to nodes
├── kube-controller     Runs controllers (replication, endpoints, etc.)
└── cloud-controller    Cloud-provider specific logic

Worker Node
├── kubelet             Ensures containers run as specified
├── kube-proxy          Maintains network rules for Services
└── container runtime   containerd, CRI-O, etc.

Abstractions
Pod              Smallest deployable unit. One or more containers.
ReplicaSet       Maintains N replicas of a pod.
Deployment       Manages ReplicaSets. Rolling updates. Rollbacks.
StatefulSet      Like Deployment but with stable identity + storage.
DaemonSet        One pod per node.
Job              Run-to-completion tasks.
CronJob          Scheduled Jobs.
Service          Stable network endpoint for a set of pods.
Ingress          HTTP/S routing rules.
ConfigMap        Non-secret configuration.
Secret           Sensitive configuration (base64-encoded).
PersistentVolume Cluster-level storage.
PVC              Request for storage by a pod.
Namespace        Virtual cluster isolation.
ServiceAccount   Identity for pod processes.
RBAC             Role-based access control.
```

---

## kubectl Essentials

### Configuration
```bash
kubectl config view                   view kubeconfig
kubectl config current-context        current context
kubectl config get-contexts           all contexts
kubectl config use-context mycontext  switch context
kubectl config set-context --current --namespace=mynamespace  set default namespace

# Shortcut: kubectx + kubens
kubectx production              switch cluster context
kubens kube-system              switch namespace
```

### Resource Operations
```bash
# Get
kubectl get pods
kubectl get pods -n kube-system       specific namespace
kubectl get pods --all-namespaces     all namespaces
kubectl get pods -o wide              more info (node, IP)
kubectl get pods -o yaml              full YAML
kubectl get pods -o json              JSON output
kubectl get pods -w                   watch for changes
kubectl get pods -l app=nginx         label selector
kubectl get pods --field-selector=status.phase=Running

# Describe (great for debugging)
kubectl describe pod mypod
kubectl describe node mynode
kubectl describe service myservice

# Create / Apply
kubectl apply -f manifest.yaml        idempotent
kubectl apply -f dir/                 apply all in directory
kubectl apply -R -f dir/              recursive
kubectl create -f manifest.yaml       fails if exists
kubectl create deployment nginx --image=nginx  imperative

# Delete
kubectl delete pod mypod
kubectl delete -f manifest.yaml
kubectl delete pod --all
kubectl delete pod -l app=nginx
kubectl delete pod mypod --grace-period=0 --force  force immediate

# Edit
kubectl edit deployment myapp         opens in $EDITOR
kubectl patch deployment myapp -p '{"spec":{"replicas":3}}'

# Labels
kubectl label pod mypod env=prod
kubectl label pod mypod env-          remove label
kubectl annotate pod mypod team=platform
```

### kubectl Output Tricks
```bash
# Custom columns
kubectl get pods -o custom-columns="NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName"

# jsonpath
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.capacity.cpu}{"\n"}{end}'

# Sort
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'

# Watch with wide
watch -n 2 'kubectl get pods -o wide'
```

---

## Pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
    version: v1.2.0
    env: production
  annotations:
    team: platform
    deployment-time: "2024-01-01"
spec:
  serviceAccountName: myapp-sa
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    runAsNonRoot: true

  initContainers:
  - name: init-db
    image: busybox:1.35
    command: ['sh', '-c', 'until nc -z db-service 5432; do sleep 2; done']

  containers:
  - name: myapp
    image: myapp:1.2.0
    imagePullPolicy: IfNotPresent   # Always | IfNotPresent | Never
    
    ports:
    - name: http
      containerPort: 3000
      protocol: TCP
    
    env:
    - name: DB_HOST
      value: "db-service"
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: environment
    
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: app-secrets
    
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "500m"
    
    volumeMounts:
    - name: config
      mountPath: /app/config
      readOnly: true
    - name: data
      mountPath: /app/data
    
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 10
      periodSeconds: 30
      timeoutSeconds: 5
      failureThreshold: 3
    
    readinessProbe:
      httpGet:
        path: /ready
        port: 3000
      initialDelaySeconds: 5
      periodSeconds: 10
    
    startupProbe:
      httpGet:
        path: /health
        port: 3000
      failureThreshold: 30
      periodSeconds: 10
    
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
  
  volumes:
  - name: config
    configMap:
      name: app-config
  - name: data
    persistentVolumeClaim:
      claimName: app-data-pvc
  
  restartPolicy: Always
  terminationGracePeriodSeconds: 30
  
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values: [myapp]
          topologyKey: kubernetes.io/hostname
  
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: myapp
```

---

## Deployments

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # can create 1 extra pod during update
      maxUnavailable: 0     # no pod allowed to be unavailable
  
  minReadySeconds: 10       # wait before considering pod ready
  revisionHistoryLimit: 5   # keep 5 old ReplicaSets for rollback
  
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.2.0
        # ... (same as pod spec)
```

### Deployment Operations
```bash
kubectl rollout status deployment myapp
kubectl rollout history deployment myapp
kubectl rollout history deployment myapp --revision=3
kubectl rollout undo deployment myapp           rollback to previous
kubectl rollout undo deployment myapp --to-revision=2
kubectl rollout pause deployment myapp          pause during update
kubectl rollout resume deployment myapp
kubectl rollout restart deployment myapp        rolling restart

kubectl scale deployment myapp --replicas=5
kubectl set image deployment myapp myapp=myapp:1.3.0   update image
kubectl set env deployment myapp KEY=value              set env var
```

---

## Services

```yaml
# ClusterIP (default) — internal only
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
  - name: http
    port: 80            # service port
    targetPort: 3000    # container port (or name)
    protocol: TCP

---
# NodePort — accessible from outside cluster on node IP
apiVersion: v1
kind: Service
metadata:
  name: myapp-nodeport
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 3000
    nodePort: 30080     # 30000-32767 (or omit for random)

---
# LoadBalancer — cloud load balancer
apiVersion: v1
kind: Service
metadata:
  name: myapp-lb
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - port: 443
    targetPort: 3000

---
# Headless — no cluster IP, DNS returns pod IPs
apiVersion: v1
kind: Service
metadata:
  name: myapp-headless
spec:
  clusterIP: None
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 3000

---
# ExternalName — DNS alias to external service
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: database.example.com
```

---

## ConfigMaps & Secrets

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  environment: "production"
  log_level: "warn"
  config.yaml: |
    server:
      port: 3000
      timeout: 30s
    database:
      pool_size: 10

---
# Secret (base64-encoded values)
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  db-password: cGFzc3dvcmQxMjM=   # echo -n 'password123' | base64
stringData:                        # auto-encoded (easier)
  api-key: "my-secret-api-key"

---
# TLS Secret
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-cert>
  tls.key: <base64-key>
```

```bash
# Create imperatively
kubectl create configmap app-config --from-literal=key=value
kubectl create configmap app-config --from-file=config.yaml
kubectl create configmap app-config --from-env-file=.env

kubectl create secret generic my-secret --from-literal=password=s3cr3t
kubectl create secret generic my-secret --from-file=ssh-key=~/.ssh/id_rsa
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key

# Encode/decode secrets
echo -n 'value' | base64
echo 'dmFsdWU=' | base64 -d
```

---

## Volumes & PersistentVolumes

```yaml
# PersistentVolume (cluster resource)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-100gi
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain    # Retain | Delete | Recycle
  storageClassName: standard
  hostPath:
    path: /data/pv                         # dev only; use cloud disk in prod

---
# PersistentVolumeClaim (namespace resource)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: standard

---
# StorageClass (dynamic provisioning)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
```

### Access Modes
```
ReadWriteOnce (RWO)   — one node read/write
ReadOnlyMany (ROX)    — many nodes read-only
ReadWriteMany (RWX)   — many nodes read/write (needs NFS/EFS/etc.)
ReadWriteOncePod      — one pod read/write (K8s 1.22+)
```

---

## Namespaces

```bash
kubectl get namespaces
kubectl create namespace staging
kubectl delete namespace staging

# Set default namespace for session
kubectl config set-context --current --namespace=production

# Resources across all namespaces
kubectl get pods --all-namespaces
kubectl get pods -A               same, shorter

# Resource quota per namespace
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    persistentvolumeclaims: "10"
    services.loadbalancers: "2"
EOF

# LimitRange (defaults per pod/container)
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:
      cpu: 100m
      memory: 128Mi
    defaultRequest:
      cpu: 50m
      memory: 64Mi
    max:
      cpu: "2"
      memory: 4Gi
EOF
```

---

## RBAC

```yaml
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production

---
# Role (namespace-scoped)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update", "patch"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-pod-reader
  namespace: production
subjects:
- kind: ServiceAccount
  name: myapp-sa
  namespace: production
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

---
# ClusterRole + ClusterRoleBinding (cluster-wide)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

```bash
# Check permissions
kubectl auth can-i create pods
kubectl auth can-i create pods --as alice
kubectl auth can-i --list                    all permissions for current user
```

---

## Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    - api.example.com
    secretName: tls-secret
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

---

## StatefulSets

For stateful applications (databases, queues, etc.):

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless   # must reference headless service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 50Gi
      storageClassName: fast
```

StatefulSet pods get stable names: `postgres-0`, `postgres-1`, `postgres-2`  
DNS: `postgres-0.postgres-headless.namespace.svc.cluster.local`

---

## DaemonSets & Jobs

```yaml
# DaemonSet — one pod per node (monitoring, logging agents)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      tolerations:
      - operator: Exists                # run on all nodes including masters
      containers:
      - name: fluentd
        image: fluentd:latest
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log

---
# Job — run to completion
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 4
  ttlSecondsAfterFinished: 3600   # auto-delete after 1h
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: migrate
        image: myapp:latest
        command: ["python", "manage.py", "migrate"]

---
# CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-cleanup
spec:
  schedule: "0 2 * * *"           cron syntax
  concurrencyPolicy: Forbid        Allow | Forbid | Replace
  failedJobsHistoryLimit: 3
  successfulJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: cleanup
            image: myapp:latest
            command: ["python", "cleanup.py"]
```

---

## Resource Management

```yaml
# Resource requests vs limits:
# request: what scheduler uses for placement
# limit: hard cap (CPU throttled, memory OOM killed)

resources:
  requests:
    memory: "128Mi"    # scheduler sees this
    cpu: "100m"        # 100 millicores = 0.1 CPU
  limits:
    memory: "256Mi"    # OOM killed if exceeded
    cpu: "500m"        # throttled if exceeded
```

```bash
# View resource usage
kubectl top pods                    requires metrics-server
kubectl top pods --sort-by=memory
kubectl top nodes

# Quality of Service classes:
# Guaranteed: requests == limits (most predictable)
# Burstable:  requests < limits (most pods)
# BestEffort: no requests or limits (evicted first)

# Node capacity vs allocatable
kubectl describe node mynode | grep -A 8 "Capacity:"
```

---

## Probes

```yaml
# Liveness: is the app alive? Restart if fails.
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
    - name: X-Health-Check
      value: k8s
  initialDelaySeconds: 15   # wait before first probe
  periodSeconds: 20          # probe every 20s
  timeoutSeconds: 5          # timeout
  successThreshold: 1        # how many to consider success
  failureThreshold: 3        # how many failures before restart

# Readiness: is it ready to receive traffic? Remove from service if fails.
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10

# Startup: for slow-starting apps. Disables liveness until passes.
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30      # 30 * 10s = 5 min to start
  periodSeconds: 10

# TCP probe (for non-HTTP services)
livenessProbe:
  tcpSocket:
    port: 5432
  initialDelaySeconds: 15
  periodSeconds: 20

# Exec probe
livenessProbe:
  exec:
    command:
    - /bin/sh
    - -c
    - pg_isready -U postgres
```

---

## Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 200Mi
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # wait 5min before scaling down
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Pods
        value: 4
        periodSeconds: 60
```

```bash
kubectl get hpa
kubectl describe hpa myapp-hpa
```

---

## Helm

```bash
# Install
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm search repo nginx
helm show values bitnami/nginx

# Install a chart
helm install my-nginx bitnami/nginx
helm install my-nginx bitnami/nginx --set replicaCount=3
helm install my-nginx bitnami/nginx -f custom-values.yaml
helm install my-nginx bitnami/nginx -n production --create-namespace

# Upgrade
helm upgrade my-nginx bitnami/nginx
helm upgrade --install my-nginx bitnami/nginx   install or upgrade

# Releases
helm list
helm list -A                 all namespaces
helm status my-nginx
helm history my-nginx

# Rollback
helm rollback my-nginx 2    rollback to revision 2

# Uninstall
helm uninstall my-nginx

# Template (dry-run)
helm template my-nginx bitnami/nginx > rendered.yaml
helm install my-nginx bitnami/nginx --dry-run

# Create a chart
helm create mychart
helm lint mychart/
helm package mychart/
```

---

## Debugging & Troubleshooting

### Pod won't start
```bash
kubectl describe pod mypod         check Events section
kubectl logs mypod                 current logs
kubectl logs mypod --previous      logs from crashed container
kubectl logs mypod -c container    specific container in multi-container pod
kubectl get events --sort-by='.lastTimestamp' -n mynamespace

# ImagePullBackOff / ErrImagePull
kubectl describe pod → check image name, registry credentials
kubectl get secret --all-namespaces | grep registry
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass

# CrashLoopBackOff
kubectl logs mypod --previous     see why it crashed
kubectl describe pod mypod        check limits, probes
# Common causes: bad env vars, missing configmap/secret, wrong command

# Pending
kubectl describe pod mypod    → "Insufficient cpu/memory" or "no nodes available"
kubectl get nodes
kubectl describe node <node>  → check Conditions, Capacity vs Allocatable
```

### Network debugging
```bash
# Run a debug pod
kubectl run debug --image=nicolaka/netshoot -it --rm -- bash
kubectl run debug --image=busybox -it --rm -- sh

# Test connectivity from inside cluster
kubectl exec -it mypod -- curl http://service-name
kubectl exec -it mypod -- nslookup service-name
kubectl exec -it mypod -- nc -zv db-service 5432

# Port-forward for local testing
kubectl port-forward pod/mypod 8080:3000
kubectl port-forward service/myservice 8080:80
kubectl port-forward deployment/myapp 8080:3000

# Check service endpoints
kubectl get endpoints myservice
kubectl describe service myservice

# Check DNS
kubectl exec -it mypod -- cat /etc/resolv.conf
kubectl exec -it mypod -- nslookup kubernetes.default
```

### Resource issues
```bash
kubectl top pods --sort-by=memory
kubectl top nodes
kubectl describe node | grep -A 10 "Allocated resources"

# Check what's consuming namespace quota
kubectl describe resourcequota -n production

# Find pending pods and why
kubectl get pods -A | grep -v Running
kubectl get pods -A --field-selector=status.phase=Pending
```

### Useful one-liners
```bash
# Get all images running in cluster
kubectl get pods -A -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' | sort | uniq

# Get pods not running
kubectl get pods -A | grep -v "Running\|Completed"

# Get pods on a specific node
kubectl get pods -A -o wide | grep node-name

# Restart all pods in deployment
kubectl rollout restart deployment myapp

# Get pod logs for all pods with a label
kubectl logs -l app=myapp --all-containers

# Delete all completed jobs
kubectl delete job $(kubectl get jobs | awk '/Completed/{print $1}')

# Watch pod status across namespaces
watch -n 2 "kubectl get pods -A | grep -v Running | grep -v Completed"
```

---

## Production Checklist

```
Resources:
  ✓ Set CPU/memory requests on all containers
  ✓ Set CPU/memory limits (memory >= requests, CPU careful)
  ✓ Configure ResourceQuotas per namespace
  ✓ Configure LimitRanges for defaults

Reliability:
  ✓ Minimum 2+ replicas for HA
  ✓ PodDisruptionBudget configured
  ✓ RollingUpdate strategy with maxUnavailable: 0
  ✓ Liveness + readiness probes configured
  ✓ startupProbe for slow-starting apps
  ✓ terminationGracePeriodSeconds adequate

Security:
  ✓ runAsNonRoot: true
  ✓ readOnlyRootFilesystem: true
  ✓ allowPrivilegeEscalation: false
  ✓ capabilities: drop ALL
  ✓ Secrets from Secret objects, not ConfigMaps
  ✓ ServiceAccount per app (not default)
  ✓ RBAC least-privilege roles
  ✓ NetworkPolicies to restrict pod communication

Operations:
  ✓ Labels: app, version, env, team
  ✓ Annotations: team contact, runbook URL
  ✓ HPA configured (scale up/down)
  ✓ PodAntiAffinity for spread across nodes
  ✓ topologySpreadConstraints
  ✓ Image pinned to digest or specific tag (not :latest)
```

```yaml
# PodDisruptionBudget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 1    # or: maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp
```
