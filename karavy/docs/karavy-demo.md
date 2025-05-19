# Karavy Demo – Creazione di un Cluster Tenant

## Requisiti

Per questa demo sono necessarie due macchine (fisiche o virtuali):

- **Prima VM**: Cluster di **management**
- **Seconda VM**: Worker node del cluster **tenant**

Devono essere sulla **stessa rete** (o comunque raggiungibili tra loro). In questa demo sono state usate VM su KVM.

---

## 1. Installazione del Cluster di Management

### 1.1 Sistema Operativo

Installare **Ubuntu 24.04 Server** e assegnare un **IP statico**.

### 1.2 Installazione di `k0s`

```bash
curl --proto '=https' --tlsv1.2 -sSf https://get.k0s.sh | sudo sh
sudo k0s install controller --single
sudo k0s start
```

### 1.3 Installazione di Helm

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
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

**Esempio configurazione IPAddressPool e L2Advertisement:**

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: karavy-demo-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.1.240-192.168.1.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: karavy-demo-advertisement
  namespace: metallb-system
```

---

## 3. Installazione Karavy-Core

```bash
git clone https://github.com/karavy/karavy-demo.git
kubectl apply -k karavy-demo/core
```

Verificare che l’operatore sia avviato nel namespace `karavy-core`.

---

## 4. Creazione del Tenant

### 4.1 Namespace e CRD

```bash
kubectl create ns tenant-1
```

Applicare il manifest (verificare IP e CIDR per evitare sovrapposizioni):

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
    KubeDefaultSvc: 10.240.0.1
    podCidr: 16
    PodNetwork: 10.120.0.0
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

---

## 5. Connessione al Cluster Tenant

```bash
kubectl get secret -n tenant-1 admin-conf -o jsonpath='{.data.tenant-1-config}' | base64 -d > /tmp/kubeconfig
export KUBECONFIG=/tmp/kubeconfig
```

---

## 6. Installazione Karavy-Worker

```bash
kubectl apply -k karavy-demo/worker
```

---

## 7. Configurazione Worker Node

### 7.1 Accesso SSH

Generare una chiave SSH e creare un secret:

```bash
kubectl create secret generic sshkey --from-file=id_rsa=~/.ssh/id_rsa --namespace=tenant-1
```

### 7.2 Manifest Worker

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

## 8. Installazione Karavy-Net

```bash
kubectl apply -k karavy-demo/net
```

---

## 9. Definizione Rotte

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