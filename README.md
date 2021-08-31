# ibm-cp4ba-demo
Installation of Cloud Pak for Business Automation on containers - Demo pattern


https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/21.0.x?topic=deployments-preparing-demo-deployment

Install oc, kubectl, podman

mkdir cp4ba-install # just a temporary directory for installation artifacts
cd cp4ba-install

### Rather curl ### wget https://github.com/IBM/cloud-pak/raw/master/repo/case/ibm-cp-automation/3.1.0/ibm-cp-automation-3.1.0.tgz
curl https://github.com/IBM/cloud-pak/raw/master/repo/case/ibm-cp-automation/3.1.0/ibm-cp-automation-3.1.0.tgz -kL -o ibm-cp-automation-3.1.0.tgz

tar -xvzf ibm-cp-automation-3.1.0.tgz

cd ibm-cp-automation/inventory/cp4aOperatorSdk/files/deploy/crs

tar -xvzf cert-k8s-21.0.2.tar

https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/21.0.x?topic=cluster-setting-up-operator-hub

https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/21.0.x?topic=hub-preparing-operator-log-file-storage

oc login --token=sha256~CWQvrSJHab-yDptBYQbFSupdhDcgBtdepLagqzIonh0 --server=https://c100-e.eu-gb.containers.cloud.ibm.com:31958

export NAMESPACE=cp4ba
oc create namespace ${NAMESPACE}
oc project ${NAMESPACE}

cd cert-kubernetes/descriptors

oc get storageclass # in both places in the operator-shared-pvc.yaml use ibmc-file-gold-gid for ROKS cluster created for CP4BA

Edit the operator-shared-pvc.yaml file and replace the <StorageClassName> and <Fast_StorageClassName> placeholders by storage classes of your choice.
Use values from your cluster.
ROKS - ibmc-file-gold-gid
### TODO sed replacement

oc apply -f operator-shared-pvc.yaml

# Wait for the PVCs to be created!
oc get pvc

# TODO consolidate
Get entitlement key from https://myibm.ibm.com/products-services/containerlibrary
export ENTITLEMENT_KEY=eyJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJJQk0gTWFya2V0cGxhY2UiLCJpYXQiOjE2MTQ5NjQzNTUsImp0aSI6IjAwZDg5NzExYTdlODQ4MzI4ZjhhY2UzZGM4NDYzYTgzIn0.6kxdf7BZn5Or3vqosw9_2KnF7FU_v7QdJH3Rlfd2waw

kubectl create secret docker-registry admin.registrykey -n ${NAMESPACE} \
   --docker-server=cp.icr.io \
   --docker-username=cp \
   --docker-password="${ENTITLEMENT_KEY}"

kubectl create secret docker-registry ibm-entitlement-key -n ${NAMESPACE} \
   --docker-username=cp \
   --docker-password="${ENTITLEMENT_KEY}" \
   --docker-server=cp.icr.io

cat <<EOF >service-account-for-demo.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ibm-cp4ba-anyuid
imagePullSecrets:
- name: "admin.registrykey"

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ibm-cp4ba-privileged
imagePullSecrets:
- name: "admin.registrykey"
EOF

oc apply -f service-account-for-demo.yaml -n ${NAMESPACE}

oc adm policy add-scc-to-user privileged -z ibm-cp4ba-privileged -n ${NAMESPACE}
oc adm policy add-scc-to-user anyuid -z ibm-cp4ba-anyuid -n ${NAMESPACE}

https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/21.0.x?topic=hub-installing-operator-catalog

cat <<EOF > ibm-operator-catalog.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ibm-operator-catalog
  namespace: openshift-marketplace
spec:
  displayName: "IBM Operator Catalog"
  publisher: IBM
  sourceType: grpc
  image: docker.io/ibmcom/ibm-operator-catalog
  updateStrategy:
    registryPoll:
      interval: 45m
EOF

oc apply -f ibm-operator-catalog.yaml

cat <<EOF > opencloud-operators-catalog.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: opencloud-operators
  namespace: openshift-marketplace
spec:
  displayName: IBMCS Operators
  publisher: IBM
  sourceType: grpc
  image: quay.io/opencloudio/ibm-common-service-catalog:latest
  updateStrategy:
    registryPoll:
      interval: 45m
EOF

oc apply -f opencloud-operators-catalog.yaml

# TODO make following operator installation scripted?

Follow step 4
In the OCP console, click Operators to open the OperatorHub, enter cp4a in the Filter by keyword box under All items.
Click Install

Follow step 5
Select your version and namespace!!!

Follow step 6
Verify the deployment by checking all of the pods are running. - All 8 pods need to be Running.
oc get pods

Also following Operators should be visible in "Installed Operators" (all in the specific namespace):
- IBM Automation Foundation Core
- IBM Automation Foundation
- IBM Cloud Pak foundational services
- IBM Cloud Pak for Business Automation

Skip step 7 - Content Collector for SAP

oc logs -f deployment/ibm-cp4a-operator -c operator # Some output should be quickly visible


https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/21.0.x?topic=deployments-installing-demo-deployment

https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/21.0.x?topic=deployment-installing-capabilities-in-operator-hub

# On Red Hat OpenShift Kubernetes Service (ROKS) only, apply the no root squash command for Db2.
oc get no -l node-role.kubernetes.io/worker --no-headers -o name | xargs -I {} \
   -- oc debug {} \
   -- chroot /host sh -c 'grep "^Domain = slnfsv4.coms" /etc/idmapd.conf || ( sed -i "s/.*Domain =.*/Domain = slnfsv4.com/g" /etc/idmapd.conf; nfsidmap -c; rpc.idmapd )'

Use the operator instance to apply a custom resource by clicking CP4BA deployment > Create ICP4ACluster (not Create Instance as in the docs).
Select desired configuration - review the YAML file FYI

Wait for TODO - Final indication of deployment

Check the Config map

