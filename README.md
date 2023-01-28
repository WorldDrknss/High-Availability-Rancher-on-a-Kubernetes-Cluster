# Requirements
- [ ] Ubuntu 20.04LTS
- [ ] Load Balancer
- [ ] MySQL or MariaDB
- [ ] Bare Metal or VMs

# Specs
We will be utilizing 6 Virtual Machines for this build.

### Kubernetes Servers: (3 Total)

| Specs | Description |
| ----------- | ----------- |
| CPU | 4 Cores |
| RAM | 8GB |
| Disk Space | 32GB |

### Kubernetes Workers: (3 Total)

| Specs | Description |
| ----------- | ----------- |
| CPU | 4 Cores |
| RAM | 8GB |
| Disk Space | 32GB |
| Additional Disk | 250GB |

# Update Ubuntu
Run the following command to make sure your Ubuntu installation is up to date.
```bash
sudo apt-get update && apt-get upgrade -y
```

# Kubernetes Installation

Kubernetes is an open-source container orchestration system for automating software deployment, scaling, and management. Google originally designed Kubernetes, but the Cloud Native Computing Foundation now maintains the project.

![Kubernetes](images/kubernetes.png)

## Install Kubernetes
Repleace the following variables
* {LOADBALANCER} = This is generally the IP address to your load balancer
* {USERNAME} = MySQL or MariaDB Username
* {PASSWORD} = MySQL or MariaDB Password
* {IP} = IP or hostname of your database server
* {PORT} = MySQL and MariaDB use 3306
* {DATABASE} = Database name

This command will spin up a kubernetes server that does not allow deployments. We will utilize worker nodes for that.
```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --node-taint CriticalAddonsOnly=true:NoExecute \
  --tls-san {LOADBALANCER} \
  --datastore-endpoint=='mysql://{USERNAME}:{PASSWORD}@tcp({IP}:{PORT})/{DATABASE}'
```

## Token
This is needed to add additional servers and workers. Replace ***{TOKEN}*** in the following commands with the output of the following:
```bash
cat /var/lib/rancher/k3s/server/token
```

## Add Additional Servers
Run the following command to add additional servers to your cluster
```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --node-taint CriticalAddonsOnly=true:NoExecute \
  --tls-san {LOADBALANCER} \
  --token={TOKEN} \
  --datastore-endpoint='mysql://{USERNAME}:{PASSWORD}@tcp({IP}:{PORT}/{DATABASE}'
```

## Add Worker Nodes
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://{LOADBALANCER}:6443 K3S_TOKEN={TOKEN} sh -
```

# Rancher Installation
Rancher, the open-source multi-cluster orchestration platform, lets operations teams deploy, manage and secure enterprise Kubernetes.

![Rancher](images/rancher.png)

## Setup kubectl
 The Kubernetes command-line tool, kubectl, allows you to run commands against Kubernetes clusters.
[https://kubernetes.io/](https://kubernetes.io/)

Kubectl is meant to run on your personal machines to manage kubernetes clusters. You can run this using WSL + Ubuntu or use the link above to configure kubectl for your environment.

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

## Configure kubectl
Copy the contents of the following command. You will need for kubectl config.
```bash
sudo cat /etc/rancher/k3s/k3s.yaml
```
Make directory to store our kubectl config.
```bash
sudo mkdir ~/.kube
```
Create our config file and past the contents from above.
```bash
sudo nano ~./kube/config
```
To save and exit nano use the following commands
```bash
ctrl+o
ctrl+x
```
Set read only permissions
```bash
sudo chmod 400 ~/.kube/config
```

## Install HELM
Helm helps you manage Kubernetes applications — Helm Charts help you define, install, and upgrade even the most complex Kubernetes application
[https://helm.sh/docs/intro/install/](https://helm.sh/docs/intro/install/)
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/stable
helm repo update
kubectl create namespace cattle-system
```

## Install Cert-Manager
Cloud native certificate management ... cert-manager is a powerful and extensible X.509 certificate controller for Kubernetes and OpenShift workloads
[https://cert-manager.io/](https://cert-manager.io/)
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.2/cert-manager.yaml
```

## Install Rancher
Replease ***{HOSTNAME}*** with the URL you intend to use to access launcher. Depending on your setup you may need to add a line to your host file that points this domain to your load balancer.
```bash
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname={HOSTNAME}
```

## Rollout Status
```bash
kubectl -n cattle-system rollout status deploy/rancher
```

## Rancher temporary password
```bash
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{.data.bootstrapPassword|base64decode}}{{ "\n" }}'
```
