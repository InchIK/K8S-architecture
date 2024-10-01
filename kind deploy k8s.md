# 需求:  1. 請以kind 架設一個3個control-plane 節點，以及4個worker 節點 。

## A. 建立 1 control-plane 跟 2 worker  in kind-cluster-config.yaml
#### (資源不足以建立3 control-plane 4 worker 以1:2替代)


<pre><code>
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker # Infra node 
  - role: worker # Application node 
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

#### kubectl label node ah-worker2 node-role=app

# 需求: 3. 安裝 MetalLB ，以L2 模式安裝，speaker 部署在 infra node上。

## A. 安裝  Nginx Ingress Controller

### wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml

### kubectl apply -f deploy.yaml

### kubectl get pods -n ingress-nginx

![image](https://github.com/InchIK/K8S-architecture/blob/master/image/k8s_ingress.png)

## B. 安裝  修改ipvs

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

<pre><code>
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.1.185.1-10.1.185.5
</code></pre>

## G. 建立L2-Advertrisement，與ipv4pool binding

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