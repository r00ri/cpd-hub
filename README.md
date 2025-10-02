# Instalasi Software Hub aka (CPD Simplified Deployment)

IBM® Software Hub adalah solusi yang dapat digunakan untuk menginstal, mengelola, dan memantau layanan Cloud Pak for Data yang berjalan pada platform Red Hat® OpenShift® Container Platform.

Langkah langkah berikut ini, untuk menginstall service service berikut
1. IBM Knowledge Catalog
2. IBM Manta Datalineage
3. MANTA Automated Datalineage
4. IBM Datastage Enterprise

# 1. Setup VM Workstation
## Prasyarat

- Sebuah virtual machine (spesifikasi minimum 4 vCPU, 8 GB Memory, 300GB Storage)
- Sistem operasi yang digunakan adalah RHEL 8 atau RHEL 9
- Virtual machine dapat mengakses github.com dan icr.io
- Virtual machine dapat mengakses alamat openshift cluster

## Prosedur
### 1.1. install packages
```bash
sudo dnf install nano wget zip
``` 
### 1.2. install docker
```bash
sudo dnf remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate  docker-engine podman runc
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo systemctl start docker
```
### 1.3. Install cpd-cli
```bash
mkdir -p /mnt/cpd && cd /mnt/cpd
wget https://github.com/IBM/cpd-cli/releases/download/v14.2.0_refresh_1/cpd-cli-linux-EE-14.2.0.tgz
tar -xvf cpd-cli-linux-EE-14.2.0.tgz
export PATH=/mnt/cpd/cpd-cli-linux-EE-14.2.0-2124:$PATH
export HOME=/mnt/cpd
cpd-cli manage restart-container
```
**Untuk Melihat Container cpd-cli**
```bash
docker container ls | grep olm-utils-play-v3
```

### 1.4. Install openshift client (oc)

**RHEL 8**
```bash
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.18.21/openshift-client-linux-amd64-rhel8-4.18.21.tar.gz
```
**RHEL 9**
```bash
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.18.21/openshift-client-linux-amd64-rhel9-4.18.21.tar.gz
```
```bash
tar -xvf openshift-client-linux-amd64-rhel8-4.18.21.tar.gz
chmod +x oc kubectl /usr/bin
mv oc kubectl /usr/bin
rm README.md
// pindahkan file kubeconfig ke folder /mnt/cpd
export KUBECONFIG=/mnt/cpd/kubeconfig
```

**Untuk Testing**
```bash
oc version
```

## Untuk Instalasi Secara Offline
1. masukkan file docker, harbor, openshift client, dan cpd-image yang sudah di archive ke VM private registry
2. setup VM private registry [guide](https://github.com/r00ri/openshift-private-registry)
3. load image olm-utils-v3
```bash
cpd-cli manage load-image --source-image=icr.io/cpopen/cpd/olm-utils-v3:5.2.1.amd64
export OLM_UTILS_IMAGE=icr.io/cpopen/cpd/olm-utils-v3:5.2.1.amd64
```
4. ketika private registry sudah ready, maka jalankan perintah berikut untuk login ke private registry

```bash
cpd-cli manage login-private-registry ${PRIVATE_REGISTRY_LOCATION} ${PRIVATE_REGISTRY_PUSH_USER} ${PRIVATE_REGISTRY_PUSH_PASSWORD}
```

5. push image ke private registry
```bash
cpd-cli manage mirror-images --components=${COMPONENTS} --release=${VERSION} --source_registry=127.0.0.1:12443 --target_registry=${PRIVATE_REGISTRY_LOCATION} --arch=${IMAGE_ARCH} --case_download=false
```

6. validasi push image

```bash
cpd-cli manage list-images --components=${COMPONENTS} --release=${VERSION} --target_registry=${PRIVATE_REGISTRY_LOCATION} --case_download=false
```

7. buat ICSP
jalankan perintah oc get imageContentSourcePolicy, jika hasilnya *No resources found*, jalankan perintah dibawah

```bash
cpd-cli manage apply-icsp --registry=${PRIVATE_REGISTRY_LOCATION}
```

8. pull image
```bash
export OLM_UTILS_IMAGE=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/olm-utils-v3:${VERSION}.amd64
```

# 2. Mengumpulkan Informasi Openshift Cluster

## Prasyarat
- VM harus bisa terhubung dengan openshift cluster
## Prosedur
### 2.1. Buat file environment variable
```bash
touch cpd_env.sh && chmod +x cpd_env.sh
nano cpd_env.sh
```
### 2.2. Isi file tersebut dengan informasi yang dibutuhkan
contoh isi file, ganti {{value}} dengan nilai yang anda inginkan
```bash
#===============================================================================
# IBM Software Hub installation variables
#===============================================================================

# ------------------------------------------------------------------------------
# Client workstation 
# ------------------------------------------------------------------------------
# Set the following variables if you want to override the default behavior of the IBM Software Hub CLI.
#
# To export these variables, you must uncomment each command in this section.

# export CPD_CLI_MANAGE_WORKSPACE=<enter a fully qualified directory>
# export OLM_UTILS_LAUNCH_ARGS=<enter launch arguments>


# ------------------------------------------------------------------------------
# Cluster
# ------------------------------------------------------------------------------

export OCP_URL={{value}}
export OPENSHIFT_TYPE=self-managed
export IMAGE_ARCH=amd64
# export OCP_USERNAME=<enter your username>
# export OCP_PASSWORD=<enter your password>
export OCP_TOKEN={{value}}
export SERVER_ARGUMENTS="--server=${OCP_URL}"
# export LOGIN_ARGUMENTS="--username=${OCP_USERNAME} --password=${OCP_PASSWORD}"
# export LOGIN_ARGUMENTS="--token=${OCP_TOKEN}"
export CPDM_OC_LOGIN="cpd-cli manage login-to-ocp ${SERVER_ARGUMENTS} ${LOGIN_ARGUMENTS}"
export OC_LOGIN="oc login ${SERVER_ARGUMENTS} ${LOGIN_ARGUMENTS}"


# ------------------------------------------------------------------------------
# Proxy server
# ------------------------------------------------------------------------------

# export PROXY_HOST=<enter your proxy server hostname>
# export PROXY_PORT=<enter your proxy server port number>
# export PROXY_USER=<enter your proxy server username>
# export PROXY_PASSWORD=<enter your proxy server password>
# export NO_PROXY_LIST=<a comma-separated list of domain names>


# ------------------------------------------------------------------------------
# Projects
# ------------------------------------------------------------------------------

export PROJECT_CERT_MANAGER=ibm-cert-manager
export PROJECT_LICENSE_SERVICE=ibm-licensing
export PROJECT_SCHEDULING_SERVICE=ibm-cpd-scheduler
# export PROJECT_IBM_EVENTS=<enter your IBM Events Operator project>
export PROJECT_PRIVILEGED_MONITORING_SERVICE=ibm-cpd-privileged
export PROJECT_CPD_INST_OPERATORS=ibm-cpd-operators
export PROJECT_CPD_INST_OPERANDS=ibm-cpd-operands
# export PROJECT_CPD_INSTANCE_TETHERED=<enter your tethered project>
# export PROJECT_CPD_INSTANCE_TETHERED_LIST=<a comma-separated list of tethered projects>



# ------------------------------------------------------------------------------
# Storage
# ------------------------------------------------------------------------------

export STG_CLASS_BLOCK=ocs-storagecluster-ceph-rbd
export STG_CLASS_FILE=ocs-storagecluster-cephfs

# ------------------------------------------------------------------------------
# IBM Entitled Registry
# ------------------------------------------------------------------------------

export IBM_ENTITLEMENT_KEY={{value}}


# ------------------------------------------------------------------------------
# Private container registry
# ------------------------------------------------------------------------------
# Set the following variables if you mirror images to a private container registry.
#
# To export these variables, you must uncomment each command in this section.

# export PRIVATE_REGISTRY_LOCATION=<enter the location of your private container registry>
# export PRIVATE_REGISTRY_PUSH_USER=<enter the username of a user that can push to the registry>
# export PRIVATE_REGISTRY_PUSH_PASSWORD=<enter the password of the user that can push to the registry>
# export PRIVATE_REGISTRY_PULL_USER=<enter the username of a user that can pull from the registry>
# export PRIVATE_REGISTRY_PULL_PASSWORD=<enter the password of the user that can pull from the registry>


# ------------------------------------------------------------------------------
# IBM Software Hub version
# ------------------------------------------------------------------------------

export VERSION=5.2.0


# ------------------------------------------------------------------------------
# Components
# ------------------------------------------------------------------------------

export COMPONENTS=ibm-licensing,scheduler,cpfs,cpd_platform,wkc,datastage_ent,datalineage,mantaflow
# export COMPONENTS_TO_SKIP=<component-ID-1>,<component-ID-2>
# export IMAGE_GROUPS=<image-group-1>,<image-group-2>
```

### 2.3. Execute file
```bash
source cpd_env.sh
```


# 3. Menyiapkan Openshift Cluster

## 3.1. Tambah Pull Secret

```bash
${CPDM_OC_LOGIN}
cpd-cli manage add-icr-cred-to-global-pull-secret --entitled_registry_key=${IBM_ENTITLEMENT_KEY}
cpd-cli manage oc get nodes
```

## 3.2. Buat Shared Cluster Component Project

```bash
oc new-project ${PROJECT_LICENSE_SERVICE}
oc new-project ${PROJECT_SCHEDULING_SERVICE}
```

## 3.3. Install Shared Cluster Component

```bash
cpd-cli manage apply-cluster-components --release=${VERSION} --license_acceptance=true --licensing_ns=${PROJECT_LICENSE_SERVICE}
oc create clusterrolebinding ibm-cpd-scheduling-system-kube-scheduler --clusterrole=system:kube-scheduler --serviceaccount=${PROJECT_SCHEDULING_SERVICE}:ibm-cpd-scheduling-operator
oc create clusterrolebinding ibm-cpd-scheduling-system-volume-scheduler --clusterrole=system:volume-scheduler --serviceaccount=${PROJECT_SCHEDULING_SERVICE}:ibm-cpd-scheduling-operator
cpd-cli manage apply-scheduler --release=${VERSION} --license_acceptance=true --scheduler_ns=${PROJECT_SCHEDULING_SERVICE}
```

## 3.4. Tuning Infrastructure

### 3.4.1. Tuning HAProxy
```bash
export TIMEOUT_SETTING=5m
sed -i -e "/timeout client/s/ [0-9].*/ ${TIMEOUT_SETTING}/" /etc/haproxy/haproxy.cfg
sed -i -e "/timeout server/s/ [0-9].*/ ${TIMEOUT_SETTING}/" /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
```
### 3.4.2. Tuning Process ID Limit
```bash
oc get kubeletconfig
```
```bash
oc apply -f - << EOF apiVersion: machineconfiguration.openshift.io/v1 kind: KubeletConfig metadata: name: cpd-kubeletconfig spec: kubeletConfig: podPidsLimit: 16384 machineConfigPoolSelector: matchExpressions: - key: pools.operator.machineconfiguration.openshift.io/worker operator: Exists EOF
```

# 4. Install Operator Yang Dibutuhkan

## 4.1. Install Openshift AI

```bash
${OC_LOGIN}
oc new-project redhat-ods-operator
cat <<EOF |oc apply -f - apiVersion: operators.coreos.com/v1 kind: OperatorGroup metadata: name: rhods-operator namespace: redhat-ods-operator EOF
export CHANNEL_VERSION=stable-2.19
cat <<EOF |oc apply -f - apiVersion: operators.coreos.com/v1alpha1 kind: Subscription metadata: name: rhods-operator namespace: redhat-ods-operator spec: name: rhods-operator channel: ${CHANNEL_VERSION} source: redhat-operators sourceNamespace: openshift-marketplace config: env: - name: "DISABLE_DSC_CONFIG" EOF
cat <<EOF |oc apply -f - apiVersion: dscinitialization.opendatahub.io/v1 kind: DSCInitialization metadata: name: default-dsci spec: applicationsNamespace: redhat-ods-applications monitoring: managementState: Managed namespace: redhat-ods-monitoring serviceMesh: managementState: Removed trustedCABundle: managementState: Managed customCABundle: "" EOF
oc get pods -n redhat-ods-operator
oc get dscinitialization
cat <<EOF |oc apply -f - apiVersion: datasciencecluster.opendatahub.io/v1 kind: DataScienceCluster metadata: name: default-dsc spec: components: codeflare: managementState: Removed dashboard: managementState: Removed datasciencepipelines: managementState: Removed kserve: managementState: Managed defaultDeploymentMode: RawDeployment serving: managementState: Removed name: knative-serving kueue: managementState: Removed modelmeshserving: managementState: Removed ray: managementState: Removed trainingoperator: managementState: Managed trustyai: managementState: Removed workbenches: managementState: Removed EOF
oc get datasciencecluster default-dsc -o jsonpath='"{.status.phase}" {"\n"}'
oc get pods -n redhat-ods-applications
```
## 4.2. Edit InferenceConfiguration
1. login ke openshift console
2. klik menu **workloads** > **configmaps** 
3. untuk project, ganti ke **redhat-ods-applications**
4. klik `inferenceservice-config` resource. kemudian, buka tab **YAML**.
5. pada bagian`metadata.annotations, tambahkan`opendatahub.io/managed: 'false'`:
```text
metadata:  
	annotations:  
		internal.config.kubernetes.io/previousKinds:  ConfigMap
		internal.config.kubernetes.io/previousNames:  inferenceservice-config  
		internal.config.kubernetes.io/previousNamespaces:  opendatahub 
		opendatahub.io/managed:  'false'
```

6. temukan baris ini
```bash
"domainTemplate":  "{{ .Name }}-{{ .Namespace }}.{{ .IngressDomain }}",
```
7. ganti dengan
```bash
"domainTemplate":  "example.com",
```

8. jangan lupa klik tombol **Save** 

# 5. Mempersiapkan Openshift Cluster
## 5.1. Health Check Cluster

```bash
cpd-cli health cluster
cpd-cli health nodes
cpd-cli health network-performance
```

## 5.2. Buat Openshift Project
```bash
oc new-project ${PROJECT_CPD_INST_OPERATORS}
oc new-project ${PROJECT_CPD_INST_OPERANDS}
```

## 5.3. Apply Permission
```bash
cpd-cli manage authorize-instance-topology --cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

# 6. Install IBM Software Hub Instance

## 6.1. Apply License
```bash
cpd-cli manage get-license --release=${VERSION} --license-type=EE
```

## 6.2. Install Instance
```bash
cpd-cli manage setup-instance --release=${VERSION} --license_acceptance=true --cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --block_storage_class=${STG_CLASS_BLOCK} --file_storage_class=${STG_CLASS_FILE} --run_storage_tests=true
```
## 6.3. Periksa Operands
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```
## 6.4. Periksa Operator dan Operands Resource
```bash 
cpd-cli health operators --operator_ns=${PROJECT_CPD_INST_OPERATORS} --control_plane_ns=${PROJECT_CPD_INST_OPERANDS}

cpd-cli health operands --control_plane_ns=${PROJECT_CPD_INST_OPERANDS}
```

## 6.5. Lihat URL Login dan Credential IBM Software
```bash
cpd-cli manage get-cpd-instance-details --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --get_admin_initial_credentials=true
```

# 7. Setup IBM Software Hub
## 7.1. Install Priviledge Monitoring
```bash
oc new-project ${PROJECT_PRIVILEGED_MONITORING_SERVICE}
cpd-cli manage apply-privileged-monitoring-service --privileged_service_ns=${PROJECT_PRIVILEGED_MONITORING_SERVICE} --cluster_components_ns=${PROJECT_SCHEDULING_SERVICE} --cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

## 7.2. Install Admission Control Webhook
```bash
cpd-cli manage install-cpd-config-ac --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
cpd-cli manage enable-cpd-config-ac --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```
## 7.3. Apply License
```bash
cpd-cli manage apply-entitlement --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --entitlement=cpd-enterprise
cpd-cli manage apply-entitlement --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --entitlement=ikc-premium
cpd-cli manage apply-entitlement --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --entitlement=data-lineage
cpd-cli manage apply-entitlement --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --entitlement=datastage
```
# 8. Install Produk
## 8.1. Enable Product Options
```bash
touch install-options.yaml
nano install-options.yaml
```
isi file tersebut dengan konten berikut
```text
################################################################################  
# IBM Knowledge Catalog parameters  
################################################################################  
custom_spec:  
	wkc:  
		enableDataQuality: True
		enableKnowledgeGraph: True
		useFDB: True
```

## 8.2. Install Produk
```bash
cpd-cli manage apply-olm --release=${VERSION} --cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} --components=${COMPONENTS}
cpd-cli manage apply-cr --release=${VERSION} --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=${COMPONENTS} --block_storage_class=${STG_CLASS_BLOCK} --file_storage_class=${STG_CLASS_FILE} --license_acceptance=true --param-file=/tmp/work/install-options.yml
```
# 9. Setup Produk
## 9.1. Enable JDBC URL

```bash
oc patch ccs ccs-cr --namespace=${PROJECT_CPD_INST_OPERANDS} --type merge --patch '{"spec": {"wdp_connect_connection_enable_jdbc_url_for_vaulting": "true", "wdp_connect_flight_enable_jdbc_url_for_vaulting": "true"}}'
oc get ccs ccs-cr --namespace=${PROJECT_CPD_INST_OPERANDS} -o template --template '{{.status.ccsStatus}}'
```
## 9.2. Enable Lineage Import
```bash
oc delete "$(oc get pods -o name | grep metadata-discovery)"
oc delete "$(oc get pods -o name | grep wkc-metadata-imports-ui)"
```

# Troubleshooting
## 1. Fitur optional tidak aktif padahal sudah define di install-options.yaml

```bash
cpd-cli manage update-cr --component=${IKC_TYPE} --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} --patch="{\"enableDataQuality\":True}"
cpd-cli manage update-cr --component=${IKC_TYPE} --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} --patch="{\"enableKnowledgeGraph\":True,\"useFDB\":True}"
```

generate a platform API key melalui IBM software hub:

1.  Log in ke IBM software hub UI.
2.  Klik gambar avatar.
3.  Klik Profile and settings.
4.  Klik API key > Generate new key.
5.  Klik Generate.
6.  Klik Copy dan simpan key di tempat tertentu.


### 1.1. Generate ZenAPI Authorization Token
```bash
echo "admin:<api_key>" | base64
export TOKEN=<base64-encoded-user-api-pair>
```

### 1.2. Start Knowlege Sync
```bash
curl -k -X POST "https://$HOST/v3/glossary_terms/admin/resync?artifact_type=category&sync_destinations=KNOWLEDGE_GRAPH" --header "Content-Type: application/json" --header "Accept: application/json" --header "Authorization: ZenApiKey ${TOKEN}" -d '{}'
curl -k -X POST "https://$HOST/v3/glossary_terms/admin/resync?artifact_type=glossary_term&sync_destinations=KNOWLEDGE_GRAPH" --header "Content-Type: application/json" --header "Accept: application/json" --header "Authorization: ZenApiKey ${TOKEN}" -d '{}'
curl -k -X POST "https://$HOST/v3/glossary_terms/admin/resync?artifact_type=classification&sync_destinations=KNOWLEDGE_GRAPH" --header "Content-Type: application/json" --header "Accept: application/json" --header "Authorization: ZenApiKey ${TOKEN}" -d '{}'
curl -k -X POST "https://$HOST/v3/glossary_terms/admin/resync?artifact_type=data_class&sync_destinations=KNOWLEDGE_GRAPH" --header "Content-Type: application/json" --header "Accept: application/json" --header "Authorization: ZenApiKey ${TOKEN}" -d '{}'
curl -k -X POST "https://$HOST/v3/glossary_terms/admin/resync?artifact_type=reference_data&sync_destinations=KNOWLEDGE_GRAPH" --header "Content-Type: application/json" --header "Accept: application/json" --header "Authorization: ZenApiKey ${TOKEN}" -d '{}'
curl -k -X POST "https://$HOST/v3/glossary_terms/admin/resync?artifact_type=policy&sync_destinations=KNOWLEDGE_GRAPH" --header "Content-Type: application/json" --header "Accept: application/json" --header "Authorization: ZenApiKey ${TOKEN}" -d '{}'
curl -k -X POST "https://$HOST/v3/glossary_terms/admin/resync?artifact_type=rule&sync_destinations=KNOWLEDGE_GRAPH" --header "Content-Type: application/json" --header "Accept: application/json" --header "Authorization: ZenApiKey ${TOKEN}" -d '{}'
```

## 2. Tidak dapat melihat knowledge graph
**solusi** : resync lineage metadata
```bash
oc edit cj catalog-api-lineage-sync-cronjob
```

termukan baris
```bash
- name: dbs_to_sync` 
-`value: '[]'
```

biarkan tetap menjadi default seperti itu

```bash
oc create job -n ${PROJECT_CPD_INST_OPERANDS} --from=cronjob/catalog-api-lineage-sync-cronjob lineage-job
```

### Install Kubernetes SMB
```bash
cat <<EOF | oc apply -f -
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # "namespace" omitted. ClusterRoles are not scoped to a namespace.
  name: ibm-zen-volumes-cluster-role
rules:
- apiGroups: [""]
  #
  # at the HTTP level, the name of the resource for accessing Secret
  # objects is "secrets"
  resources: ["persistentvolumes"]
  verbs: ["create", "get", "list", "patch", "update", "watch", "delete", "use"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ibm-zen-volumes-cluster-role-binding
subjects:
- kind: ServiceAccount
  name: ibm-zen-operator-serviceaccount
  namespace: ibm-common-services    # The namespace where the IBM Cloud Pak foundational services are installed
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ibm-zen-volumes-cluster-role
EOF
```
