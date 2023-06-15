# Setup a Kubernetes Study Lab on MacBook/iMac (M1)

***In this Article:***
- [Overview](#overview)
- [What's my Rig for this tutorial and Recommendation](#whats-my-rig-for-this-tutorial-and-recommendation)
- [Tools used to Deploy the K8s Cluster](#tools-used-to-deploy-the-k8s-cluster)
- [Additional topics in this tutorial: Cilium, Hubble, Istio Service Mesh, CIS-Kube-bench](#additional-topics-in-this-tutorial-install-cilium-hubble-istio-service-mesh-cis-benchmarks-with-kube-bench)
- [Is there single push Automated Script](#is-there-single-push-automated-script)
- [Installation Steps for a Multi-node Kubernetes Cluster](#installation-steps-for-a-multi-node-kubernetes-cluster)
  - [Download and setup Kubespray](#a-download-and-setup-kubespray)
  - [Deploy the Kubernetes Cluster](#b-deploy-the-kubernetes-cluster)
  - [Setup CLI, Access the Cluster & Label the Nodes](#c-setup-cli-access-the-cluster--label-the-nodes)
  - [Remove kube-proxy and Install Cilium CNI Plugin](#d-remove-kube-proxy-and-install-cilium-cni-plugin)
  - [Access the cluster with Kubernetes Dashboard](#e-access-the-cluster-with-kubernetes-dashboard)
  - [Install Hubble - Network, Service & Security Observability for Kubernetes](#f-install-hubble---network-service--security-observability-for-kubernetes)
- [Installation Steps for ISTIO Service Mesh](#installation-steps-for-istio-service-mesh)
  - [Install Istio](#a-install-istio)
  - [Install Add-ons: Kiali, Jaegar, Prometheus & Grafana](#b-install-addons-kiali-jaegar-prometheus--grafana)
  - [Install the sample Booking Micro-service App to test the Mesh](#c-install-the-sample-booking-micro-service-app-to-test-the-mesh)
  - [Uninstall the Booking App](#d-uninstall-the-booking-app-if-needed)
- [Run CIS Kubernetes Benchmarks to secure the cluster](#run-cis-kubernetes-benchmarks-to-secure-the-cluster)


## Overview

This tutorial will guide you with the steps and commands to setup a K8s Lab  with a High touch composable solution to make it more interactive with Kubernetes components and architecture. We will use Kubespray which is composition of Ansible playbooks and commonly used to deploy a self-managed production grade Kubernetes cluster. It also let's you customize the setup to incorporate best practices. I'll cover some of them in this tutorial.

Alternatively, there are several Low-touch, Turn-key solutions for setting up a quick K8s enviornment for Dev/StudyLab purpose.

**Low-touch alternatives:**
- [Kind](https://external.ink?to=https://kind.sigs.k8s.io/)
- [MicroK8s](https://external.ink?to=https://microk8s.io/)
- [MiniKube](https://external.ink?to=https://minikube.sigs.k8s.io/docs/)

**Turn-key production grade alternative:**
- [Rancher](https://external.ink?to=https://www.rancher.com/)

**If you are preparing for the Official Linux Foundation Certification Exam, then below Cloud based solution is widely used.**
- [Killercoda](https://external.ink?to=https://killercoda.com/)

## What's my Rig for this tutorial and Recommendation

- iMac (M1) / MacOS Ventura 13.4
- Minimum 8GB RAM (Base Study Lab with capacity available to Host OS) 
- Recommended 16GB (With Istio, ArgoCD, Crossplane, Deploy test application etc)

## Tools used to Deploy the K8s Cluster
- **[MultiPass - Ubuntu](https://external.ink?to=https://multipass.run/):** To provision virtual nodes for a multi-node cluster
- **[Kubespray](https://github.com/kubernetes-sigs/kubespray):** To deploy the kubernetes cluster on the virtual nodes
- **[Ansible](https://www.ansible.com/overview/how-ansible-works):** Tool used by kubespray to deploy the K8s cluster
- Shell scripting 

## Additional topics in this tutorial: Install Cilium, Hubble, Istio Service Mesh, CIS Benchmarks with Kube-bench
- **[Cilium](https://external.ink?to=https://cilium.io/get-started/):** Cilium is an open source, cloud native solution for providing, securing, and observing network connectivity between workloads, fueled by the revolutionary Kernel technology eBPF.
- **[Hubble](https://external.ink?to=https://github.com/cilium/hubble#what-is-hubble):** Hubble is a fully distributed networking and security observability platform for cloud native workloads. It is built on top of Cilium and eBPF to enable deep visibility into the communication and behavior of services as well as the networking infrastructure in a completely transparent manner.
- **[Istio Service Mesh](https://external.ink?to=https://istio.io/latest/about/service-mesh/):** A service mesh is a dedicated infrastructure layer that you can add to your applications. It allows you to transparently add capabilities like observability, traffic management, and security, without adding them to your own code. Application developers can focus on Business logic than worrying about the infrastructure layer logic.
- **[Kube-Bench](https://external.ink?to=https://github.com/aquasecurity/kube-bench):** kube-bench is a tool that checks whether Kubernetes is deployed securely by running the checks documented in the CIS Kubernetes Benchmark.


## Is there single push Automated Script?
The goal of this tutorial is to be interactive and walk with the steps involved to create a K8s cluster. The low level commands can then be easily programmed for one-push fully automated tool with inputs and validations.

## Installation Steps for a Multi-node Kubernetes Cluster

### A] Download and setup Kubespray

**Step 1:**  We will use the iMac/Macbook as the Ansible control node.  Set the desired  environment parameters.

```bash
# Set desired Project Home
export PROJECT_HOME=$HOME/Projects/k8s
# No Change
export MULTIPASS_KEY=$HOME/.ssh/multipass.key
# Set Ubuntu version. 
export UBUNTU_REL=22.10
# Set desired number of nodes for multi-node K8s cluster. Set 2 if 8GB RAM.
# Best Practice: Odd number to prevent Split Brain 
NODE_COUNT=2
# Desired multipass node name prefix
export NODE_PREFIX=k8snode
# Compute for the Virtual nodes
CPU=2
MEM=3G
# Disk can be expanded if required but can't be shrinked
DISK=10G
# Set stable K8 version
export K8S_VERSION=v1.27.2
# Set Desired cluster name. Avoid using "." in cluster name for eg homelab.local. Cilium installation will fail.
export CLUSTER_NAME=homelab
```

**Step 2:** Install Multipass and copy the SSH private key to home directory of current Mac user.

```bash
brew install --cask multipass
```

```bash
sudo cp /var/root/Library/Application\ Support/multipassd/ssh-keys/id_rsa $MULTIPASS_KEY && sudo chown $USER $MULTIPASS_KEY
``` 

**Step 3:** Provision the virtual nodes for K8s cluster.

##### :memo: Create the virtual nodes
```bash
count=1
until [ $count -gt $NODE_COUNT ]
do
  multipass launch $UBUNTU_REL -n "$NODE_PREFIX$count" -c $CPU -m $MEM -d $DISK
 ((count=count+1))
done 
```

##### :memo: List the virtual nodes
```bash
multipass list
```

**Step 4:** Download Kubespray and install the pre-requisite packages with python-pip

##### :memo: Python may already exist on MacOS
```bash
brew list python
```

##### :memo: ...but if not then install it
```bash
brew install python3
brew postinstall python3
```

##### :memo: Create Project home
```bash
mkdir -p $PROJECT_HOME && cd $PROJECT_HOME
```

##### :memo: Download Kubespray
```bash
git clone https://github.com/kubernetes-sigs/kubespray.git && cd kubespray 
```

##### :memo: Install dependency packages
```bash
pip3 install -r requirements.txt 
```
##### :memo: Refresh the login shell environment
```bash
bash
```

**Step 5:** Set the Ansible Private key and Playbook log file

```bash
cd $PROJECT_HOME/kubespray

sed -i '' '/^\[defaults\]/a \
private_key_file = '"$MULTIPASS_KEY"' \
' ansible.cfg

sed -i '' '/^\[defaults\]/a \
log_path = '"$PROJECT_HOME"'/kubespray/playbook.log \
' ansible.cfg
```

**Step 6:** Generate the Ansible Host Inventory

```bash
cd $PROJECT_HOME/kubespray && \
cp -r inventory/sample inventory/k8cluster && \
declare -a IPS=( $(multipass list --format csv |tail -n +2 |cut -d "," -f3 | xargs) ) && \
CONFIG_FILE=inventory/k8cluster/hosts.yml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```

**Step 7:** Update the Ubuntu nodes with Ansible play

```bash
cat > initialsetup.yml <<EOF
- hosts: all
  become: true
  tasks:   
    - name: Update and upgrade apt packages
      apt:
        upgrade: yes
        update_cache: yes
    - name: Install pip, net-tools   
      apt:
        name: 
          - pip
          - net-tools
        state: present
        update_cache: true
EOF
```
```bash
ansible-playbook -i inventory/k8cluster/hosts.yml ./initialsetup.yml -e ansible_user=ubuntu
```

### B] Deploy the Kubernetes Cluster

**Step 1:** Configure the K8s cluster parameters

##### :memo: Backup original files
```bash
cd $PROJECT_HOME/kubespray/inventory/k8cluster/group_vars/k8s_cluster  && cp k8s-cluster.yml k8s-cluster.yml.BAK ; cp addons.yml addons.yml.BAK
```

##### :memo: Set K8s version
```bash
sed -i '' "s~^kube_version:.*~kube_version: $K8S_VERSION~g" k8s-cluster.yml
```

##### :memo: Set cluster name
```bash
sed -i '' "s~^cluster_name:.*~cluster_name: $CLUSTER_NAME~g" k8s-cluster.yml
```

##### :memo: Set network plugin to cni for now. We will install Cilium separately to be more interactive with the plugin.
```bash
sed -i '' 's~^kube_network_plugin:.*~kube_network_plugin: cni~g' k8s-cluster.yml
```

##### :memo: Enable encryption of secret data at rest in etcd
```bash
sed -i '' 's~^kube_encrypt_secret_data:.*~kube_encrypt_secret_data: true~g' k8s-cluster.yml
```

##### :memo: Install Kubernetes dashboard
```bash
sed -i '' "s~^# dashboard_enabled:.*~dashboard_enabled: true~g" addons.yml
```

##### :memo: Install Helm client
```bash
sed -i '' "s~^helm_enabled:.*~helm_enabled: true~g" addons.yml
```

##### :memo: Install metrics server
```bash
sed -i '' "s~^metrics_server_enabled:.*~metrics_server_enabled: true~g" addons.yml
```

##### :memo: Enable NTP sync
```bash
cd $PROJECT_HOME/kubespray/inventory/k8cluster/group_vars/all && cp all.yml all.yml.BAK
sed -i '' "s~^ntp_enabled:.*~ntp_enabled: true~g" all.yml
```

**Step 2:** Deploy the K8s cluster

##### :memo: Run the deployment
```bash
cd $PROJECT_HOME/kubespray && ansible-playbook -i ./inventory/k8cluster/hosts.yml ./cluster.yml -e ansible_user=ubuntu -b --become-user=root 
```

##### :memo: Expected Output

```
PLAY RECAP  
localhost                  : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
node1                      : ok=709  changed=144  unreachable=0    failed=0    skipped=1204 rescued=0    ignored=5   
node2                      : ok=607  changed=115  unreachable=0    failed=0    skipped=1055 rescued=0    ignored=2   

kubernetes/control-plane : Joining control plane node to the cluster.
16.50s
download : download_container | Download image if required
11.64s
kubernetes/control-plane : kubeadm | Initialize first master
8.56s
etcd : reload etcd
8.17s
download : download_file | Download item
7.27s
kubernetes/preinstall : Ensure NTP package
6.57s
kubernetes-apps/ansible : Kubernetes Apps | Start Resources
6.26s
kubernetes/preinstall : Install packages requirements
5.88s
download : download_container | Download image if required
5.84s
download : download_container | Download image if required
5.78s
etcd : Configure | Check if etcd cluster is healthy
5.38s
container-engine/crictl : download_file | Download item
5.36s
download : download_container | Download image if required
5.27s
download : download_container | Download image if required
5.24s
etcd : Configure | Ensure etcd is running
5.22s
download : download_container | Download image if required
4.75s
download : download_file | Download item
4.41s
kubernetes-apps/metrics_server : Metrics Server | Apply manifests
4.31s
container-engine/containerd : download_file | Download item
4.18s
download : download_container | Download image if required
3.72s
```

### C] Setup CLI, Access the Cluster & Label the Nodes

**Step 1:** Setup Kube config to authenticate and access the cluster

##### :memo: Connect to the first node
```bash
multipass shell "$NODE_PREFIX"1
```

##### :memo: Create kubectl shortcut aliases
```bash
cat >> ~/.bashrc <<EOF
alias k="kubectl"
alias ksn="kubectl config set-context --current --namespace"
EOF
bash
``` 

##### :memo: Copy the cluster admin config
```bash
mkdir .kube && sudo cp /etc/kubernetes/admin.conf .kube/config && sudo chown ubuntu .kube/config
```

**Step 2:** List the nodes and Label worker nodes as per your preference. 

```bash
for i in $(k get nodes -o custom-columns=NAME:.metadata.name --no-headers)
do
  k label node $i node-role.kubernetes.io/worker=worker  
done
k get nodes
```

### D] Remove kube-proxy and Install Cilium CNI Plugin

**Step 1:** Uninstall Kube-proxy

##### :memo: Delete the daemonset and configmap
```bash 
ksn kube-system && k delete ds kube-proxy && k delete cm kube-proxy
```

##### :memo: Dump the iptables rules to remove the Kube proxy rules 
##### :bulb: Run on all nodes from host ansible control node
```bash
cd $PROJECT_HOME/kubespray && \
ansible all -m shell -a "sudo iptables-save | grep -v KUBE | sudo iptables-restore" -i ./inventory/k8cluster/hosts.yml -e ansible_user=ubuntu 
```

##### :memo: Connect back to the K8s node1
```bash
multipass shell "$NODE_PREFIX"1
```

##### :memo: Download Cilium CLI
```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/master/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

##### :memo: Install cilium with Kube-proxy replacement set to strict
```bash
cilium install -n kube-system --version=v1.13.3 \
   --helm-set k8sServiceHost=localhost \
   --helm-set k8sServicePort=6443 \
   --helm-set kubeProxyReplacement=strict 
```

##### :memo: Install hubble
```bash
cilium hubble enable --ui \
   --helm-set hubble.ui.enabled=true \
   --helm-set hubble.ui.service.type=NodePort \
   --helm-set hubble.relay.service.type=NodePort \
   --helm-set hubble.peerService.clusterDomain=homelab
```

##### :memo: Verify Cilium pods are running
```bash
k -n kube-system get pods -l app.kubernetes.io/part-of=cilium -w
```

##### :memo: Check cilium status
```bash
cilium status
```

##### :memo: Output should look like below

```
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    disabled (using embedded mode)
 \__/¯¯\__/    Hubble Relay:       OK
    \__/       ClusterMesh:        disabled

```

##### :memo: Verify kube proxy replacement is set to strict and list the service map table
```bash
CILIUM_POD=$(k -n kube-system get pods -l k8s-app=cilium -o custom-columns=NAME:.metadata.name --no-headers | head -1)
k exec -it $CILIUM_POD  -- cilium status | grep KubeProxyReplacement
k exec -it $CILIUM_POD  -- cilium service list
```

##### :memo: Verify Nodes are in Ready state
```bash
k get nodes
```

### E] Access the cluster with Kubernetes Dashboard

**Step 1:** Configure Kubernetes Dashboard

##### :memo: Create service account
```bash
ksn kube-system && k create sa dashboard 
```

##### :memo: Assign cluster-admin role to the service account
```bash
k create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard
```

##### :memo: Create the secret object of type token for the service account
```bash
k apply -f - <<EOF
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: dashboard
  annotations:
    kubernetes.io/service-account.name: dashboard
EOF
```

##### :memo:  Expose the dashboard service as NodePort to access from outside the cluster network.
```bash
k patch svc kubernetes-dashboard  --type='json' -p '[{"op":"replace","path":"/spec/type","value":"NodePort"}]'
export DASHBOARD_PORT=$(k get svc kubernetes-dashboard -o jsonpath='{.spec.ports[].nodePort}')
export DASHBOARD_HOST=$(k get po -l k8s-app=kubernetes-dashboard -o jsonpath='{.items[0].status.hostIP}')
```

**Step 2:** Access the Dashboard in Browser of your host Desktop/Laptop

##### :memo: Open the URL in your browser
```bash
echo https://$DASHBOARD_HOST:$DASHBOARD_PORT
```

##### :memo: Login with the generated Token
```bash
k get secret dashboard -o jsonpath="{.data.token}" | base64 --decode
```

### F] Install Hubble - Network, Service & Security Observability for Kubernetes

##### :memo: Download Hubble
```bash
export HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
HUBBLE_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then HUBBLE_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-${HUBBLE_ARCH}.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-${HUBBLE_ARCH}.tar.gz /usr/local/bin
rm hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
```

##### :memo: Verify Hubble is working through CLI
```bash
cilium hubble port-forward & 
sleep 5
hubble status
```

##### :memo: Expected Output
```
Healthcheck (via localhost:4245): Ok
Current/Max Flows: 4,065/8,190 (49.63%)
Flows/s: 40.13
Connected Nodes: 2/2
```

##### :memo: Verify Hubble is working through UI dashboard 
```bash
export HUBBLE_PORT=$(k get svc hubble-ui -o jsonpath='{.spec.ports[].nodePort}')
export HUBBLE_HOST=$(k get po -l k8s-app=hubble-ui -o jsonpath='{.items[0].status.hostIP}')
echo "http://$HUBBLE_HOST:$HUBBLE_PORT/?namespace=kube-system"
```

## Installation Steps for ISTIO Service Mesh

### A] Install Istio

##### :memo: Install the Istio Helm repo
```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts && \
helm repo update 
```

##### :memo: Istio components
```bash
helm install istio-base istio/base -n istio-system --create-namespace
```

##### :memo: This will take few seconds to minutes
```bash
helm install istiod istio/istiod -n istio-system --wait
```

##### :memo: This output will fail because the service is created as loadbalancer and there is no external IP available. Ignore for now
```bash
helm install istio-ingressgateway istio/gateway -n istio-system --wait
```

##### :memo: Verify the gateway is running
```bash
ksn istio-system && k get ingressgateway 
```

### B] Install Addons: Kiali, Jaegar, Prometheus & Grafana

##### :memo: Install the add-ons
```bash
k apply -f https://raw.githubusercontent.com/istio/istio/release-1.18/samples/addons/kiali.yaml
k apply -f https://raw.githubusercontent.com/istio/istio/release-1.18/samples/addons/prometheus.yaml
k apply -f https://raw.githubusercontent.com/istio/istio/release-1.18/samples/addons/grafana.yaml
```

##### :memo: Verify the deployment is ready
```bash
k get deploy -n istio-system -o wide
```

##### :memo: Change the service type to NodePort to make it accessible from the Host Laptop/Desktop
```bash
k patch svc istio-ingressgateway --type='json' -p '[{"op":"replace","path":"/spec/type","value":"NodePort"}]'
k patch svc kiali --type='json' -p '[{"op":"replace","path":"/spec/type","value":"NodePort"}]'
k patch svc prometheus --type='json' -p '[{"op":"replace","path":"/spec/type","value":"NodePort"}]'
k patch svc grafana --type='json' -p '[{"op":"replace","path":"/spec/type","value":"NodePort"}]'
```

### C] Install the sample Booking Micro-service App to test the Mesh

##### :memo: Create the namespace and label for the sidecar injection
```bash
k create ns servicemesh && \
k label namespace servicemesh istio-injection=enabled && \
ksn servicemesh
```

##### :memo: Install the booking app and gateway
```bash
k apply -f https://raw.githubusercontent.com/istio/istio/release-1.17/samples/bookinfo/platform/kube/bookinfo.yaml
```
```bash
k get pods
```
```bash
k apply -f https://raw.githubusercontent.com/istio/istio/release-1.18/samples/bookinfo/networking/bookinfo-gateway.yaml
```

##### :memo: Generate the URLs
```bash
export KIALI_PORT=$(k -n istio-system get svc kiali -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
export PROM_PORT=$(k -n istio-system get svc prometheus -o jsonpath='{.spec.ports[].nodePort}')
export GRAFANA_PORT=$(k -n istio-system get svc grafana -o jsonpath='{.spec.ports[].nodePort}')
export BOOK_APP_PORT=$(k -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export INGRESS_HOST=$(k get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')
export BOOK_APP_URL=$INGRESS_HOST:$BOOK_APP_PORT
export KIALI_URL=$INGRESS_HOST:$KIALI_PORT
export PROMETHEUS_URL=$INGRESS_HOST:$PROM_PORT
export GRAFANA_URL=$INGRESS_HOST:$GRAFANA_PORT
echo "http://$BOOK_APP_URL/productpage"
echo "http://$KIALI_URL"
echo "http://$PROMETHEUS_URL"
echo "http://$GRAFANA_URL"
```

##### :memo: Trigger synthetic load on the URL to generate the traffic graphs in Kiali and monitoring data in Grafana
```bash
for i in $(seq 1 100); do curl -s -o /dev/null "http://$BOOK_APP_URL/productpage";done
```

### D] Uninstall the Booking App if needed.

```bash
ksn servicemesh && \
k delete -f https://raw.githubusercontent.com/istio/istio/release-1.17/samples/bookinfo/platform/kube/bookinfo.yaml && \
k delete -f https://raw.githubusercontent.com/istio/istio/release-1.18/samples/bookinfo/networking/bookinfo-gateway.yaml
```

## Run CIS Kubernetes Benchmarks to secure the cluster

##### :memo: Download latest kube-bench tool
```bash
curl -s \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  https://api.github.com/repos/aquasecurity/kube-bench/releases/latest |grep "browser_download_url.*arm64.deb" |
  cut -d : -f 2,3 |
  tr -d \" |
  wget -i - 
```

##### :memo: Install kube-bench tool
```bash
KBENCH_NAME=`curl -s \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  https://api.github.com/repos/aquasecurity/kube-bench/releases/latest |grep -o "name.*arm64.deb" |
  cut -d : -f 2,3 |
  tr -d " \""` && \
sudo apt -y install ./$KBENCH_NAME
```

##### :memo: Run the CIS benchmark
```bash
kube-bench  
```