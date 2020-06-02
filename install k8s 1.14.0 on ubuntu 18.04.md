## install k8s 1.14.0 on ubuntu 18.04 

### 1 Install docker 19.03.x

- set Tsinghua mirror source (/etc/apt/source.list)

  ```
  # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic main restricted universe multiverse
  
  deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
  
  # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
  
  deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
  
  # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
  
  deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
  
  # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
  ```

- install docker ce

  ```
  sudo apt-get remove docker docker-engine docker.io
  sudo apt-get update
  sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
  sudo apt-key fingerprint 0EBFCD88
  sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
  sudo apt-get update
  sudo apt-get install -y docker-ce
  ```

  https://www.scaleway.com/en/docs/how-to-install-docker-community-edition-ubuntu-bionic-beaver/



### 2 Prepare for k8s installation

- close mem swapoff

  ```
  echo "vm.swappiness = 0" >> /etc/sysctl.conf
  swapoff -a
  setenforce 0
  ```

- (option) kubeadm reset  (if reinstall again)

  ```
  kubeadm reset
  ```

- (option) reset  (if reinstall again)

  ```
  systemctl stop kubelet
  rm -rf /var/lib/cni/
  rm -rf /var/lib/kubelet/*
  rm -rf /etc/cni/
  ifconfig cni0 down
  ifconfig flannel.1 down
  ip link delete cni0
  ip link delete flannel.1
  systemctl start kubelet
  ```

  

### 3 Install kubelet kubeadm kubectl

- set k8s source list 

  ```
  apt-get update && apt-get install -y apt-transport-https
  curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
  ```

- set `/etc/apt/sources.list.d/kubernetes.list`

  ```
  deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
  ```

- apt-get update 

- install

  ```
  apt-get install -y kubelet=1.14.0-00 kubeadm=1.14.0-00 kubectl=1.14.0-00
  ```

- hold version NOT for update

  ```
  apt-mark hold kubelet kubeadm kubectl
  ```

  


### 4 Install k8s master

```
kubeadm init --image-repository=registry.aliyuncs.com/google_containers --pod-network-cidr=10.245.0.0/16 --apiserver-advertise-address=<192.168.1.194> --kubernetes-version=v1.14.0 --node-name=<prd-k8s-master> --ignore-preflight-errors=all
```

- assign privileges to root

```
mkdir -p /root/.kube
cp -rf /etc/kubernetes/admin.conf /root/.kube/config
export KUBECONFIG=/root/.kube/config
```

- install network plugin

```
kubectl apply -f ./flannel/kube-flannel.yml
```

- (option) set master to worker node

```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

- install dashboard 

  - create secret

    ```
    kubectl create secret generic kubernetes-dashboard-certs --from-file=./dashboard/crt/dashboard.key --from-file=./dashboard/crt/dashboard.crt -n kube-system
    ```

  - create dashboard	

    ```
    kubectl apply -f ./dashboard/kubernetes-dashboard.yaml	
    ```

  - create account

    ```
    kubectl create serviceaccount dashboard-admin -n kube-system
    kubectl create clusterrolebinding cluster-dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
    ```

  - create heapster plugin

    ```
    kubectl apply -f ./dashboard/heapster/
    ```

- install ingress

  ```
  kubectl apply -f ./ingress/mandatory.yaml
  kubectl apply -f ./ingress/service-nodeport.yaml
  ```

- install nvidia plugin

  ```
  kubectl apply -f ./nvidia/nvidia-device-plugin.yml
  ```
- install selinux tools

  ```
  sudo apt install policycoreutils selinux-utils selinux-basics
  ```

- set selinux `/etc/selinux/config` to disabled presistent 

  ```
  SELINUX=disabled
  ```
- disable swap partition comment line contain swap in file `/etc/fstab`

- verify master has been finished successfully

### 5 Install k8s worker and add node to cluster

- execute step 1~3

- fetch join command 

  ```
  kubeadm token create --print-join-command
  ```

- execute join command 

- install network plugin (ON master)

  ```
  kubectl apply -f ./flannel/kube-flannel.yml
  ```
  
- install selinux tools

  ```
  sudo apt install policycoreutils selinux-utils selinux-basics
  ```

- set selinux `/etc/selinux/config` to disabled presistent 

  ```
  SELINUX=disabled
  ```
  
- disable swap partition comment line contain swap in file `/etc/fstab`

- verify worker has been joined to cluster 

  

### 6 remove node from cluster

- on master

  ```
  kubectl drain k8s-gpu-node1 --delete-local-data --force --ignore-daemonsets
  kubectl delete node k8s-gpu-node1
  ```

- on slave

  - reset

    ```
    kubeadm reset
    rm /etc/kubernetes -rf
    ```

  - fetch join command 

    ```
    kubeadm token create --print-join-command
    ```

  - execute join command 

### 7 others

- query token shell 

  ```
  kubectl get secret -n kube-system | grep dashboard-admin | awk '{print $1}' | xargs kubectl describe -n kube-system secret
  ```

- hostnamectl

  ```
  sudo hostnamectl set-hostname prd-k8s-master
  ```

  

