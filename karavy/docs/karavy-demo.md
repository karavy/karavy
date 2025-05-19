# Karavy Demo – Creazione di un Cluster Tenant

## Requisiti

Per questa demo sono necessarie due macchine (fisiche o virtuali):

* **Prima VM**: Cluster di **management**
    - CPU: 2
    - RAM: 4GB
    - Disk: 40GB
* **Seconda VM**: Worker node del cluster **tenant**
    - CPU: 2
    - RAM: 4GB
    - Disk: 40GB
* **Terza VM**: (Opzionale) Installazione del server NFS per i volumi etcd del cluster **tenant**. Nel caso in cui si disponga già di storage condiviso disponibile per il cluster di management, si può utilizzare quello.
    - CPU: 1
    - RAM: 2GB
    - Disk: 40GB

Le macchine devono essere sulla **stessa rete** (o comunque raggiungibili tra loro senza limitazione di porte o protocolli). In questa demo sono state usate VM su KVM.

---

## 1. Installazione del Cluster di Management

### 1.1 Sistema Operativo

Installare **Ubuntu 24.04 Server** e assegnare un **IP statico** ad entrambe le macchine. Durante l'installazione attivare anche il server ssh

### 1.2 Installazione di `k0s`

Collegarsi alla vm che ospiterà il cluster di management e installare k0s

```bash
curl --proto '=https' --tlsv1.2 -sSf https://get.k0s.sh | sudo sh
sudo k0s install controller --single
sudo k0s start
```

### 1.3 Installare e configurare kubectl per l'accesso al cluster di management

```bash
sudo -i
snap install kubectl --classic
mkdir /home/<user>/.kube
k0s kubeconfig admin > /home/<user>/.kube/config
chown -R <user>:<group> /home/<user>/.kube
exit
```

### 1.4 Installazione di Helm

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

### 1.5 Installazione di storage condiviso per i volumi di etcd

#### 1.5.1 Installazione e configurazione server NFS

Installare il server NFS e creare la directory che conterrà i volumi kubernetes

```bash
sudo apt install nfs-server
sudo mkdir /home/nfsdata
```

Creare la condivisione NFS

```bash
sudo -i
echo '/home/nfsdata        <cluster_network>(no_root_squash,rw,sync,no_subtree_check)' >> /etc/exports
exportfs -va
```

---

## 2. Installazione Componenti del Cluster

### 2.1 Cert-Manager

```bash
helm repo add jetstack https://charts.jetstack.io --force-update
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.17.2 --set crds.enabled=true
```

### 2.2 MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml
```

#### 2.2.1 Configurazione IPAddressPool e L2Advertisement

Configurare Metallb. Nei manifest seguenti si utilizza la modalità layer2 

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: karavy-demo-pool
  namespace: metallb-system
spec:
  addresses:
    - <first_pool_ip>-<last_pool_ip>
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: karavy-demo-advertisement
  namespace: metallb-system
```

## 2.3 Installazione provider CSI nel cluster di management

Nel cluster di management installare il driver csi

```bash
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs --namespace kube-system --version 4.11.0 --set kubeletDir="/var/lib/k0s/kubelet"
```

Sempre nel cluster di management creare la storage class per il provider applicando il seguente manifest

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: <nfs_server_ip>
  share: /home/nfsdata
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
mountOptions:
  - nfsvers=4.1
```

---

## 3. Installazione Karavy-Core

Nel cluster di management installare i componente core per la gestione dei tenant

```bash
sudo apt install -y git
git clone https://github.com/karavy/karavy-demo.git
kubectl apply -k karavy-demo/core
```

Verificare che l’operatore sia avviato nel namespace `karavy-core`.

---

## 4. Creazione del Tenant

### 4.1 Namespace e CRD

Nel cluster di management creare il namespace relativo al nuovo tenant

```bash
kubectl create ns tenant-1
```

Applicare nel cluster di management il manifest del nuovo tenant (verificare IP e CIDR per evitare sovrapposizioni):

```yaml
apiVersion: tenants.karavy.io/v1
kind: K8sTenant
metadata:
  labels:
    tenant-name: tenant-1
  name: tenant-1
  namespace: tenant-1
spec:
  kubeMasterDomain: cluster.local
  tenantApiServer:
    additionalArgs:
      - --v=10
    antiAffinity: true
    certificateDurationHours: 8760
    keycloak:
      enabled: false
    replicas: 1
  tenantControllerManager:
    antiAffinity: true
    certificateDurationHours: 8760
    replicas: 1
  tenantDNSServer:
    certificateDurationHours: 8760
  tenantEtcdServer:
    additionalArgs:
      - --initial-advertise-peer-urls=$(URI_SCHEME)://$(HOSTNAME).$(SERVICE_NAME).$(K8S_NAMESPACE):2380
    antiAffinity: true
    certificateDurationHours: 8760
    replicas: 3
    storageClassName: nfs-csi
    storageSize: 10Gi
  tenantNetwork:
    cniApplication:
      type: calico
      version: v3.29.2
    kubeDefaultSvc: 10.240.0.1
    podCidr: 16
    podNetwork: 10.120.0.0
    serviceCidr: 16
    serviceNetwork: 10.240.0.0
    tenantKubeDNSIP: 10.240.0.10
    tenantKubeDomain: tenant-1.local
  tenantScheduler:
    antiAffinity: true
    certificateDurationHours: 8760
    replicas: 1
  tenantVersions:
    tenantCrioVersion: v1.31.5
    tenantEtcdVersion: v3.5.18
    tenantKubeVersion: v1.31.5
  usersCertificates:
    certificateDurationHours: 8760
  winNodeSupport: true
  workerCertificates:
    certificateDurationHours: 8760
```

### 4.2 Connessione al Cluster Tenant

Per prima cosa estrarre il secret relativo al nuovo tenant

```bash
kubectl get secret -n tenant-1 admin-conf -o jsonpath='{.data.tenant-1-config}' | base64 -d > /tmp/kubeconfig
export KUBECONFIG=/tmp/kubeconfig
```

Eseguendo il comando di elencazione dei pod

```bash
kubectl get pods -A
```

Il risultato visualizzato mostrerà l'operator di Calico e il dns in Pending, in attesa cioè di un nodo worker per eseguire il carico.

```bash
NAMESPACE         NAME                               READY   STATUS    RESTARTS   AGE
kube-system       forwarder-dns-fd89bcbd5-l8wbq      0/1     Pending   0          86m
kube-system       forwarder-dns-fd89bcbd5-vbnwz      0/1     Pending   0          86m
tigera-operator   tigera-operator-64ff5465b7-8r8mm   0/1     Pending   0          86m
```
---

## 5. Installazione Karavy-Worker

Dal repository scaricato in precedenza, eseguire il comando per installare il gestore dei nodi worker. Al momento è possibile 
creare nodi linux basati su Ubuntu 24.04. Il cluster supporta anche nodi Microsoft, la documentazione relativa verrà pubblicata a breve.

```bash
kubectl apply -k karavy-demo/worker
```

---

## 6. Configurazione Worker Node

### 6.1 Accesso SSH

Generare una chiave SSH e creare un secret:

```bash
kubectl create secret generic sshkey --from-file=id_rsa=~/.ssh/id_rsa --namespace=tenant-1
```

### 6.2 Manifest Worker

```yaml
apiVersion: tenants.karavy.io/v1
kind: K8sTenantWorker
metadata:
  name: tenant-1-worker-01
  namespace: tenant-1
spec:
  tenantName: tenant-1
  hostname: worker-01
  hostIP: 172.25.58.100
  sshKeySecret: sshkey
  operatingSystem: ubuntu2404
  sshPort: 22
```

---

## 7. Installazione Karavy-Net

```bash
kubectl apply -k karavy-demo/net
```

### 7.1 Definizione Rotte

```yaml
apiVersion: net.karavy.io/v1
kind: KaravyRouting
metadata:
  labels:
    app.kubernetes.io/name: karavy-net
    app.kubernetes.io/managed-by: kustomize
  name: karavy-tenant-1-routing
  namespace: karavy-net
spec:
  tenantName: tenant-1
  mainServiceCidr: 10.96.0.0/12
  mainPodCidr: 10.244.0.0/16
  mainCNIType: userdefined
  mainClusterNodes:
    - mainClusterNodeType: controlplane
      name: kavary-main-cluster
      servicePriority: 10
  tenantServiceCidr: 10.240.0.0/16
  tenantPodCidr: 10.120.0.0/16
  tenantCNIType: calico
  tenantClusterNodes:
    - ip: 192.168.3.216
      name: worker-01
      servicePriority: 10
      sshKey: sshkey
      sshPort: 22
```
