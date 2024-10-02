# 需求:  1. 請以kind 架設一個3個control-plane 節點，以及4個worker 節點 。

## A. 建立 1 control-plane 跟 2 worker  in kind-cluster-config.yaml
#### (資源不足以建立3 control-plane 4 worker 以1:2替代)


<pre><code>
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  #disableDefaultCNI: true
nodes:
- role: control-plane
  extraPortMappings:
    - containerPort: 30090   
      hostPort: 30090
      protocol: TCP
    - containerPort: 31717  
      hostPort: 31717
      protocol: TCP
- role: worker
- role: worker
</code></pre>

## B. 建立集群 

#### kind create cluster --config kind-cluster-config.yaml

## C. 檢查集群

#### kubectl get nodes

![image](https://github.com/InchIK/K8S-architecture/blob/master/image/k8s_1.png)

# 需求: 2. 節點分為2群角色或功能

## A.  為node打上labels

當叢集建立完成後，您可以使用 kubectl 給 Infra nodes 和 Application nodes 打上 labels。

先列出所有節點，識別 worker 節點的名稱：

#### kubectl get nodes

![image](https://github.com/InchIK/K8S-architecture/blob/master/image/k8s_2.png)

worker 節點名稱是 ha-worker, ha-worker2，然後打上 labels：

為 Infra nodes 打上 label:

#### kubectl label node ha-worker node-role=infra

為 Application nodes 打上 label:

#### kubectl label node ha-worker2 node-role=app
![image](https://github.com/InchIK/K8S-architecture/blob/master/image/k8s_labels.png)

# 需求: 3. 安裝 MetalLB ，以L2 模式安裝，speaker 部署在 infra node上。

## A. 安裝  Nginx Ingress Controller

### wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml

### kubectl apply -f deploy.yaml

### kubectl get pods -n ingress-nginx

![image](https://github.com/InchIK/K8S-architecture/blob/master/image/k8s_ingress.png)

## B. 安裝  修改ipvs

### kubectl edit configmap -n kube-system kube-proxy

加入 ipvs設定
<pre><code>
    kind: KubeProxyConfiguration
    mode: "ipvs"
    ipvs:
      strictARP: true
</code></pre>

![image](https://github.com/InchIK/K8S-architecture/blob/master/image/k8s_ipvs.png)

## C. 下載 yaml

### curl -O https://raw.githubusercontent.com/metallb/metallb/refs/heads/main/config/manifests/metallb-native.yaml
### curl -O https://raw.githubusercontent.com/metallb/metallb/main/config/manifests/metallb-frr.yaml

## D. 安裝metallb

### kubectl create ns metallb-system

### kubectl apply -f metallb-native.yaml -n metallb-system

### kubectl apply -f metallb-frr.yaml -n metallb-system

## E. speaker 部署在 infra node上

<pre><code>
kubectl patch daemonset speaker -n metallb-system --patch '{
  "spec": {
    "template": {
      "spec": {
        "nodeSelector": {
          "node-role": "infra"
        }
      }
    }
  }
}'
</code></pre>

![image](https://github.com/InchIK/K8S-architecture/blob/master/image/k8s_path_infra.png)

## F. 建立IP-Pool

### kubectl apply -f ip_pool.yaml

<pre><code>
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.1.5.220-10.1.5.224
</code></pre>

## G. 建立L2-Advertrisement，與ipv4pool binding

### kubectl apply -f L2advertisement.yaml

<pre><code>
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool 
</code></pre>

![image](https://github.com/InchIK/K8S-architecture/blob/master/image/k8s_ippool.png)

## H. 驗證LoadBlance是否要到IP

#### Create Nginx
<pre><code>
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: docker.io/nginx:latest
        ports:
        - containerPort: 80 
</code></pre>

#### Create Service
<pre><code>
apiVersion: v1
kind: Service
metadata:
  name: nginx2
  labels:
    app: nginx
spec:
  selector:
    app: nginx
  ports:
  - name: nginx-port
    protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer
</code></pre>

![image](https://github.com/InchIK/K8S-architecture/blob/master/image/k8s_get_ip.png)

# 需求: 4. 安裝 Prometheus, node exporter, kube-state-metrics 在infra node 節點上，Prometheus 收集node exporter, kube-state-metrics的效能數據

## HELM 新增 prometheus-community
#### helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
#### helm repo update
#### helm show values prometheus-community/kube-prometheus-stack > values.yaml
## A. 修改values.yaml Prometheus, node exporter, kube-state-metrics 加上親和性，僅佈署在ha-worker
<pre><code>
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: node-role
                operator: In
                values:
                  - infra
</code></pre>

## B. values.yaml註解掉 grafana
grafana:
  enabled: false

## C. 安裝 prometheus  
#### helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring -f values.yaml

![image](https://github.com/InchIK/K8S-architecture/blob/master/image/k8s_node-exporter.png)
![image](https://github.com/InchIK/K8S-architecture/blob/master/image/k8s_prometheus.png)

# 需求: 5. 安裝 Grafana 在kind叢集外，以docker或podman 執行，datasouce指向 Prometheus，並呈現3個效能監控儀表板。

### docker pull grafana/grafana

### docker run -d --name=grafana -p 3000:3000 grafana/grafana

## datasouce指向 Prometheus

![image](https://github.com/InchIK/K8S-architecture/blob/master/image/k8s_datasource.png)

## Grafana 面板

![image](https://github.com/InchIK/K8S-architecture/blob/master/image/k8s_grafana.png)

## CPU 使用率

### 說明: 用來監控 Node 的 CPU 資源是否有過度使用或不足，以避免單個節點成為性能瓶頸。
![image](https://github.com/InchIK/K8S-architecture/blob/master/image/k8s_g_cpu.png)

## 記憶體使用率

### 說明: 測量記憶體的使用情況，檢查是否有記憶體資源耗盡的風險，防止 OOM（Out Of Memory）錯誤發生。
![image](https://github.com/InchIK/K8S-architecture/blob/master/image/k8s_g_memory.png)

## 磁碟 I/O 使用率

### 說明: 磁碟 I/O 是系統效能的關鍵因素，過度使用會導致應用程式讀寫延遲，應該保持在合理的範圍內。
![image](https://github.com/InchIK/K8S-architecture/blob/master/image/k8s_g_io.png)

## Pod 數量

### 說明: 監控集群中運行的 Pod 數量，確保集群中有足夠的資源來運行 Pod，並及時處理 Pod 的失效或過多的情況。
![image](https://github.com/InchIK/K8S-architecture/blob/master/image/k8s_g_pod.png)

## etcd 目標數量

### 說明: 檢查 etcd 叢集的 leader 狀態，確保叢集中至少有一個節點是 leader 並處於正常狀態。
![image](https://github.com/InchIK/K8S-architecture/blob/master/image/k8s_g_etcd_number.png)

## etcd 物件存儲大小

### 說明: 檢查 etcd 資料庫大小，以防止資料庫過大導致效能下降或儲存不足。
![image](https://github.com/InchIK/K8S-architecture/blob/master/image/k8s_g_etcd_size.png)

### 以上要觀察CPU Throttling不足，因為 CPU Throttling 是由於容器資源限制導致的情況，需要專門的指標來監控這一現象。

## CPU Throttling現象 

### 可利用container_cpu_cfs_throttled_periods_total指標進行觀察，因本次Lab為kind及資源不足，以下為本人服務公司所使用grafana面板用來觀測CPU Throttling

![image](https://github.com/InchIK/K8S-architecture/blob/master/image/k8s_CPU_Throttling.png)

# 需求: 6. 請部署一個容器應用程式在application node，建立一個hpa物件以cpu 使用率到達50%為條件，最多擴充到10個pod。

![image](https://github.com/InchIK/K8S-architecture/blob/master/image/k8s_hpa.png)
