# Environment
- Controller
    - vCores: 4
    - memory: 10G
    - os: Ubuntu 24.04.1 LTS
- worker1:
    - vCores: 2
    - memory: 4G
    - os: Ubuntu 24.04.1 LTS
- worker2:
    - vCores: 2
    - memory: 4G
    - os: Ubuntu 22.04.5 LTS


# Container Runtimes
> reference: https://kubernetes.io/docs/setup/production-environment/container-runtimes/

## CRI-O Installation
> 這次使用 CRI-O 作為我的 Container Runtime <br>
> reference: https://github.com/cri-o/packaging/blob/main/README.md#usage

```shell
# Define the CRI-O version
CRIO_VERSION=v1.32

# Install the dependencies for adding repositories
apt-get update
apt-get install -y software-properties-common curl

# Add the CRI-O repository
curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/ /" | tee /etc/apt/sources.list.d/cri-o.list

# Install the packages
apt-get update
apt-get install -y cri-o

# Start CRI-O
systemctl enable --now crio.service
```

# Kubernetes
## Prerequisite
### Swap Configruation
> reference: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#swap-configuration

```shell
swapoff -a

vim /etc/fstab
# 註解或刪除 Swap 那行，避免開機後系統自動將 Swap 啟用
mount -a
```

### Network Configuration
```shell
# 允許網橋（bridge）接收網路層的流量過濾功能
# 加載相關模塊
sudo modprobe br_netfilter
# 建立相關設定
echo 'br_netfilter' | sudo tee /etc/modules-load.d/br_netfilter.conf
# 立即採用設定
sudo sysctl -p

# IP轉發和網橋過濾相關的設定
# 使通過網橋的流量可以被iptables的規則處理
echo "1" | tee /proc/sys/net/bridge/bridge-nf-call-iptables
# 開啟IP轉發
echo "net.ipv4.ip_forward = 1" | tee -a /etc/sysctl.conf
# 寫入設定 確保每次開機可以直接運行
echo "net.bridge.bridge-nf-call-iptables = 1" | tee -a /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" | tee -a /etc/sysctl.conf
# 立即載入設定
sudo sysctl -p
```

### Packages Installation
> reference: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

```shell
apt-get update
apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list

apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
systemctl enable --now kubelet
```

## Init cluster
> 確認所有 kubernetes nodes 的 /etc/hosts，讓所有 node 都認識彼此

#### Kubernetes Controller
> 如果有多個 container runtimes 在同一台機器上，需要指定 container runtime endpoint<br>
> reference: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-runtime
```shell
kubeadm init \
--cri-socket=unix:///var/run/crio/crio.sock \
--control-plane-endpoint=controller \
--apiserver-advertise-address=192.168.1.201 \
--pod-network-cidr=10.10.0.0/16 \
--v=5
```

初始化成功後，會得到以下資訊
```txt
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join controller:6443 --token xxxxxx.xxxxxxxxxxxxxxxx \
	--discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx \
	--control-plane 

Then you can join any number of worker nodes by running the following on each as root:

  kubeadm join controller:6443 --token xxxxxx.xxxxxxxxxxxxxxxx \
	--discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx 
```

#### Kubernetes Worker
```shell
kubeadm join controller:6443 --token xxxxxx.xxxxxxxxxxxxxxxx \
	--discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx \
    --cri-socket=unix:///var/run/crio/crio.sock \
    --v=5
```

### Troubleshoot init
還原 kubeadm
```shell
kubeadm reset -f
```

重新建立加入 cluster 的 token
```shell
# 在主节点上运行以下命令，检查当前的 token 是否仍然有效
kubeadm token list

# 重新生成 Token
kubeadm token create

# 确认 discovery-token-ca-cert-hash 值
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
openssl rsa -pubin -outform der 2>/dev/null | \
sha256sum | awk '{print $1}'

# 重新运行 kubeadm join，替换为新生成的 token 和哈希值。
kubeadm join controller:6443 --token xxxxxx.xxxxxxxxxxxxxxxx \
	--discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx \
    --cri-socket=unix:///var/run/crio/crio.sock
```

## Network Addons
> reference: https://kubernetes.io/docs/concepts/cluster-administration/addons/
> 擇一安裝就可以了

### flannel
> https://github.com/flannel-io/flannel#deploying-flannel-manually
>> kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml <br>
>> If you use custom podCIDR (not 10.244.0.0/16) you first need to download the above manifest and modify the network to match your one <br>
>> 在我的 cluster 使用的 podCIDR 是 10.10.0.0/16

```yml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    k8s-app: flannel
    pod-security.kubernetes.io/enforce: privileged
  name: kube-flannel
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: flannel
  name: flannel
  namespace: kube-flannel
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: flannel
  name: flannel
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    k8s-app: flannel
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-flannel
---
apiVersion: v1
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.10.0.0/16",
      "EnableNFTables": false,
      "Backend": {
        "Type": "vxlan"
      }
    }
kind: ConfigMap
metadata:
  labels:
    app: flannel
    k8s-app: flannel
    tier: node
  name: kube-flannel-cfg
  namespace: kube-flannel
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: flannel
    k8s-app: flannel
    tier: node
  name: kube-flannel-ds
  namespace: kube-flannel
spec:
  selector:
    matchLabels:
      app: flannel
      k8s-app: flannel
  template:
    metadata:
      labels:
        app: flannel
        k8s-app: flannel
        tier: node
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      containers:
      - args:
        - --ip-masq
        - --kube-subnet-mgr
        command:
        - /opt/bin/flanneld
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: EVENT_QUEUE_DEPTH
          value: "5000"
        image: docker.io/flannel/flannel:v0.26.3
        name: kube-flannel
        resources:
          requests:
            cpu: 100m
            memory: 50Mi
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
            - NET_RAW
          privileged: false
        volumeMounts:
        - mountPath: /run/flannel
          name: run
        - mountPath: /etc/kube-flannel/
          name: flannel-cfg
        - mountPath: /run/xtables.lock
          name: xtables-lock
      hostNetwork: true
      initContainers:
      - args:
        - -f
        - /flannel
        - /opt/cni/bin/flannel
        command:
        - cp
        image: docker.io/flannel/flannel-cni-plugin:v1.6.0-flannel1
        name: install-cni-plugin
        volumeMounts:
        - mountPath: /opt/cni/bin
          name: cni-plugin
      - args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        command:
        - cp
        image: docker.io/flannel/flannel:v0.26.3
        name: install-cni
        volumeMounts:
        - mountPath: /etc/cni/net.d
          name: cni
        - mountPath: /etc/kube-flannel/
          name: flannel-cfg
      priorityClassName: system-node-critical
      serviceAccountName: flannel
      tolerations:
      - effect: NoSchedule
        operator: Exists
      volumes:
      - hostPath:
          path: /run/flannel
        name: run
      - hostPath:
          path: /opt/cni/bin
        name: cni-plugin
      - hostPath:
          path: /etc/cni/net.d
        name: cni
      - configMap:
          name: kube-flannel-cfg
        name: flannel-cfg
      - hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
        name: xtables-lock
```

### calico
> https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart

1. Install the Tigera Calico operator and custom resource definitions.
    > kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/tigera-operator.yaml <br>

2. Install Calico by creating the necessary custom resource.
    > kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/custom-resources.yaml <br>
    >> If you use custom podCIDR (not 10.244.0.0/16) you first need to download the above manifest and modify the network to match your one <br>
    >> 在我的 cluster 使用的 podCIDR 是 10.10.0.0/16
    ```yaml
    # This section includes base Calico installation configuration.
    # For more information, see: https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.Installation
    apiVersion: operator.tigera.io/v1
    kind: Installation
    metadata:
    name: default
    spec:
    # Configures Calico networking.
    calicoNetwork:
        ipPools:
        - name: default-ipv4-ippool
        blockSize: 26
        cidr: 10.10.0.0/16
        encapsulation: VXLANCrossSubnet
        natOutgoing: Enabled
        nodeSelector: all()

    ---

    # This section configures the Calico API server.
    # For more information, see: https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.APIServer
    apiVersion: operator.tigera.io/v1
    kind: APIServer
    metadata:
    name: default
    spec: {}
    ```

## Ingress controller
> https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/

### Nginx ingress controller
> https://kubernetes.github.io/ingress-nginx/deploy/#bare-metal-clusters <br>
> kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0/deploy/static/provider/baremetal/deploy.yaml
