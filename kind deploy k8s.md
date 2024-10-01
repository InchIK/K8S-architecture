## 需求:  1. 請以kind 架設一個3個control-plane 節點，以及4個worker 節點 。

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

## D.  為node打上labels

當叢集建立完成後，您可以使用 kubectl 給 Infra nodes 和 Application nodes 打上 labels。

先列出所有節點，識別 worker 節點的名稱：

#### kubectl get nodes

![image](https://github.com/InchIK/K8S-architecture/blob/master/image/k8s_2.png)

假設 worker 節點名稱是 kind-worker, kind-worker2，然後打上 labels：

為 Infra nodes 打上 label:

#### kubectl label node kind-worker node-role=infra

為 Application nodes 打上 label:

#### kubectl label node kind-worker2 node-role=app


