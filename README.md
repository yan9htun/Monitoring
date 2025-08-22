# Method of Procedure: Monitoring Namespace Setup

## 1. Introduction

This document provides a step-by-step guide for setting up the `monitoring` namespace in a Kubernetes cluster. This includes deploying Prometheus for metrics collection, Alertmanager for alerting, `kube-state-metrics` for cluster-level metrics, and `node-exporter` for node-level metrics.

## 2. Prerequisites

*   A running Kubernetes cluster.
*   `kubectl` installed and configured to connect to your cluster.
*   An NFS server accessible from the Kubernetes cluster for the `nfs-client` StorageClass.

## 3. Procedure

### 3.1. Create the `monitoring` Namespace

Create a new namespace for the monitoring components:
```bash
kubectl create namespace monitoring
```

### 3.2. Create the `nfs-client` StorageClass

Create the `nfs-client` StorageClass to provide persistent storage for Prometheus.

**File: `sc-nfs-client.yaml`**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: nfs-storage
reclaimPolicy: Retain
volumeBindingMode: Immediate
parameters:
  archiveOnDelete: "false"
```

**Apply the YAML:**
```bash
kubectl apply -f sc-nfs-client.yaml
```

### 3.3. Deploy Prometheus

#### 3.3.1. Create Service Accounts and RBAC

Create the necessary service accounts, cluster roles, and cluster role bindings for Prometheus.

**File: `prometheus-rbac.yaml`**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-k8s
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-k8s-discovery
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-k8s-discovery
subjects:
- kind: ServiceAccount
  name: prometheus-k8s
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: prometheus-k8s-discovery
  apiGroup: rbac.authorization.k8s.io
```

**Apply the YAML:**
```bash
kubectl apply -f prometheus-rbac.yaml
```

#### 3.3.2. Create ConfigMaps

Create the `prometheus-config` and `prometheus-rules` ConfigMaps.

**File: `cm-prometheus-config.yaml`**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s

    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
          - targets: ['localhost:9090']

      - job_name: 'alertmanager'
        static_configs:
          - targets: ['alertmanager:9093']

      - job_name: 'node-exporter'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app]
            action: keep
            regex: node-exporter
          - action: labelmap
            regex: __meta_kubernetes_pod_label_(.+)
          - source_labels: [__meta_kubernetes_pod_node_name]
            target_label: instance

      - job_name: 'calico-felix'
        kubernetes_sd_configs:
          - role: endpoints
            namespaces:
              names: ['kube-system']
        relabel_configs:
          - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
            action: keep
            regex: calico-felix-metrics;http-metrics

      - job_name: 'kube-state-metrics'
        kubernetes_sd_configs:
          - role: endpoints
            namespaces:
              names: ['monitoring']
        relabel_configs:
          - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
            action: keep
            regex: kube-state-metrics;http-metrics

      - job_name: 'kubernetes-apiservers'
        kubernetes_sd_configs:
        - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: default;kubernetes;https

      - job_name: 'kubernetes-cadvisor'
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        kubernetes_sd_configs:
        - role: node
        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor

      - job_name: 'lakehouse-pods'
        kubernetes_sd_configs:
        - role: pod
          namespaces:
            names: ['lakehouse']
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name

    alerting:
      alertmanagers:
        - static_configs:
            - targets:
                - 'alertmanager:9093'

    rule_files:
      - /etc/prometheus/rules/alerts.yml
```

**File: `cm-prometheus-rules.yaml`**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-rules
  namespace: monitoring
data:
  alerts.yml: |
    groups:
    - name: example
      rules:
      - alert: ExampleAlert
        expr: vector(1)
        for: 1m
        labels:
          severity: 'critical'
        annotations:
          summary: 'This is an example alert'
```

**Apply the YAMLs:**
```bash
kubectl apply -f cm-prometheus-config.yaml
kubectl apply -f cm-prometheus-rules.yaml
```

#### 3.3.3. Deploy Prometheus StatefulSet and Service

**File: `sts-prometheus.yaml`**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: prometheus
  namespace: monitoring
spec:
  serviceName: "prometheus"
  replicas: 2
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus-k8s
      containers:
      - name: prometheus
        image: prom/prometheus:v2.51.2
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus"
        - "--web.listen-address=:9090"
        ports:
        - containerPort: 9090
          name: web
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1
            memory: 2Gi
        volumeMounts:
        - name: prometheus-config
          mountPath: /etc/prometheus
        - name: prometheus-storage
          mountPath: /prometheus
        - name: prometheus-rules
          mountPath: /etc/prometheus/rules
      volumes:
      - name: prometheus-config
        configMap:
          name: prometheus-config
      - name: prometheus-rules
        configMap:
          name: prometheus-rules
  volumeClaimTemplates:
  - metadata:
      name: prometheus-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "nfs-client"
      resources:
        requests:
          storage: 8Gi
```

**File: `svc-prometheus.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
spec:
  ports:
  - name: web
    port: 9090
    targetPort: web
  selector:
    app: prometheus
  type: NodePort
```

**Apply the YAMLs:**
```bash
kubectl apply -f sts-prometheus.yaml
kubectl apply -f svc-prometheus.yaml
```

### 3.4. Deploy Alertmanager

#### 3.4.1. Create Service Account and ConfigMap

**File: `sa-alertmanager.yaml`**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: alertmanager
  namespace: monitoring
```

**File: `cm-alertmanager-config.yaml`**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      resolve_timeout: 5m
    route:
      group_by: ['alertname']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 1h
      receiver: 'web.hook'
    receivers:
    - name: 'web.hook'
      webhook_configs:
      - url: 'http://example.com:5001/'
```

**Apply the YAMLs:**
```bash
kubectl apply -f sa-alertmanager.yaml
kubectl apply -f cm-alertmanager-config.yaml
```

#### 3.4.2. Deploy Alertmanager Deployment and Service

**File: `deploy-alertmanager.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alertmanager
  template:
    metadata:
      labels:
        app: alertmanager
    spec:
      serviceAccountName: alertmanager
      containers:
      - name: alertmanager
        image: prom/alertmanager:v0.27.0
        args:
        - "--config.file=/etc/alertmanager/alertmanager.yml"
        ports:
        - containerPort: 9093
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
        volumeMounts:
        - name: alertmanager-config
          mountPath: /etc/alertmanager
      volumes:
      - name: alertmanager-config
        configMap:
          name: alertmanager-config
```

**File: `svc-alertmanager.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: alertmanager
  namespace: monitoring
spec:
  ports:
  - port: 9093
    targetPort: 9093
  selector:
    app: alertmanager
```

**Apply the YAMLs:**
```bash
kubectl apply -f deploy-alertmanager.yaml
kubectl apply -f svc-alertmanager.yaml
```

### 3.5. Deploy `kube-state-metrics`

#### 3.5.1. Create Service Account and RBAC

**File: `rbac-kube-state-metrics.yaml`**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-state-metrics
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kube-state-metrics
rules:
- apiGroups: [""]
  resources:
  - nodes
  - pods
  - services
  - resourcequotas
  - replicationcontrollers
  - limitranges
  - persistentvolumeclaims
  - persistentvolumes
  - namespaces
  - endpoints
  verbs: ["list", "watch"]
- apiGroups: ["extensions"]
  resources:
  - daemonsets
  - deployments
  - replicasets
  - ingresses
  verbs: ["list", "watch"]
- apiGroups: ["apps"]
  resources:
  - statefulsets
  - deployments
  - daemonsets
  - replicasets
  verbs: ["list", "watch"]
- apiGroups: ["batch"]
  resources:
  - cronjobs
  - jobs
  verbs: ["list", "watch"]
- apiGroups: ["autoscaling"]
  resources:
  - horizontalpodautoscalers
  verbs: ["list", "watch"]
- apiGroups: ["authentication.k8s.io"]
  resources:
  - tokenreviews
  verbs: ["create"]
- apiGroups: ["authorization.k8s.io"]
  resources:
  - subjectaccessreviews
  verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kube-state-metrics
subjects:
- kind: ServiceAccount
  name: kube-state-metrics
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: kube-state-metrics
  apiGroup: rbac.authorization.k8s.io
```

**Apply the YAML:**
```bash
kubectl apply -f rbac-kube-state-metrics.yaml
```

#### 3.5.2. Deploy `kube-state-metrics` Deployment and Service

**File: `deploy-kube-state-metrics.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-state-metrics
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kube-state-metrics
  template:
    metadata:
      labels:
        app: kube-state-metrics
    spec:
      serviceAccountName: kube-state-metrics
      containers:
      - name: kube-state-metrics
        image: bitnami/kube-state-metrics:2.16.0-debian-12-r4
        ports:
        - containerPort: 8080
          name: http-metrics
        - containerPort: 8081
          name: telemetry
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

**File: `svc-kube-state-metrics.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: kube-state-metrics
  namespace: monitoring
spec:
  ports:
  - name: http-metrics
    port: 8080
    targetPort: http-metrics
  - name: telemetry
    port: 8081
    targetPort: telemetry
  selector:
    app: kube-state-metrics
```

**Apply the YAMLs:**
```bash
kubectl apply -f deploy-kube-state-metrics.yaml
kubectl apply -f svc-kube-state-metrics.yaml
```

### 3.6. Deploy `node-exporter`

#### 3.6.1. Create Service Account

**File: `sa-node-exporter.yaml`**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: node-exporter
  namespace: monitoring
```

**Apply the YAML:**
```bash
kubectl apply -f sa-node-exporter.yaml
```

#### 3.6.2. Deploy `node-exporter` DaemonSet

**File: `ds-node-exporter.yaml`**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      serviceAccountName: node-exporter
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.8.1
        ports:
        - containerPort: 9100
          hostPort: 9100
          name: metrics
        resources:
          requests:
            cpu: 100m
            memory: 50Mi
          limits:
            cpu: 100m
            memory: 50Mi
```

**Apply the YAML:**
```bash
kubectl apply -f ds-node-exporter.yaml
```

## 4. Verification

After deploying all the components, you can verify that everything is running correctly by checking the status of the pods and services:

```bash
kubectl -n monitoring get all,pvc -o wide
```

You can also access the Prometheus UI by port-forwarding the Prometheus service:

```bash
kubectl -n monitoring port-forward svc/prometheus 9090
```

Then, open your browser and navigate to `http://localhost:9090`. You should be able to see the Prometheus UI and the targets you have configured.
