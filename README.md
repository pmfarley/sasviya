# SAS Viya on ROSA - deployment notes

[TOC]



Local mirror inside SAS network: https://gitlab.sas.com/gtp/viya4-deployments/aws-rosa-validation

Copies of all YAML manifests etc. can be found in the `backup` folder.



## Connect to environment

SSH (Hans)

```shell
ssh -i "/drives/c/Users/gerhje/OneDrive - SAS/Arbeit/ssh_keys/sasviya-ocp-rosa.id" ec2-user@18.221.95.98
```

OCP Web Console (`kubeadmin` / <password from mail>)

```
https://console-openshift-console.apps.rosa.sasviya-5adf.mh2a.p3.openshiftapps.com/
```



## Cluster topology

```shell
# show SAS labels
printf "%-50s %-30s\n" "NodeName" "workload.sas.com/class"
for node in $(kubectl get nodes -o name); do
  for label in $(kubectl get node ${node##*/} -o json | jq -c '.metadata.labels."workload.sas.com/class"'); do
	printf "%-50s %-30s\n" ${node##*/} ${label}
  done
done

NodeName                                           workload.sas.com/class        
ip-10-0-0-235.us-east-2.compute.internal           "cas"                         
ip-10-0-0-248.us-east-2.compute.internal           "stateless"                   
ip-10-0-0-250.us-east-2.compute.internal           "stateful"                    
ip-10-0-0-56.us-east-2.compute.internal            "compute"                     
ip-10-0-0-94.us-east-2.compute.internal            null                          
ip-10-0-0-99.us-east-2.compute.internal            null                          

# show taints
kubectl get nodes -o='custom-columns=NodeName:.metadata.name,TaintKey:.spec.taints[].key,TaintValue:.spec.taints[].value,TaintEffect:.spec.taints[*].effect'

NodeName                                   TaintKey                 TaintValue   TaintEffect
ip-10-0-0-235.us-east-2.compute.internal   workload.sas.com/class   cas          NoSchedule
ip-10-0-0-248.us-east-2.compute.internal   workload.sas.com/class   stateless    NoSchedule
ip-10-0-0-250.us-east-2.compute.internal   workload.sas.com/class   stateful     NoSchedule
ip-10-0-0-56.us-east-2.compute.internal    workload.sas.com/class   compute      NoSchedule
ip-10-0-0-94.us-east-2.compute.internal    <none>                   <none>       <none>
ip-10-0-0-99.us-east-2.compute.internal    <none>                   <none>       <none>
```



## Deploy OpenLDAP server

```shell
oc new-project openldap
oc create sa openldap-sa

oc adm policy add-scc-to-user nonroot -n openldap -z openldap-sa

oc apply -f ~/yaml/openldap-deploy.yaml 
oc get deploy,svc,pods
```

Test

```shell
kubectl exec -it deploy/viya4-openldap-server --insecure-skip-tls-verify -- ldapsearch -H ldap://viya4-openldap-service.openldap.svc.cluster.local:389 -b "dc=example,dc=org" -x -D "cn=admin,dc=example,dc=org" -w "lnxsas"
```

Accounts:

* sasadm / lnxsas
* demo01 / lnxsas
* demo02 / lnxsas



## Create RWX Storage fileshare and test

Fileshare (`sas-efs-shared-pvc`) used by SAS, requires `viya4` namespace. Note: this needs to be in place before the deployment starts.

```shell
kubectl apply -f ~/yaml/create-pvc-shared-data.yaml
oc get pv,pvc

oc apply -f ~/yaml/test-rwx-storage.yaml
# check the log for any errors
oc logs job/rwx-storage-test

kubectl delete -f ~/yaml/create-pvc-shared-data.yaml
```



## Additional tooling

```shell
# jq and others
sudo yum install -y mlocate vim wget git jq zip unzip tmux

# yq
wget https://github.com/mikefarah/yq/releases/download/v4.42.1/yq_linux_amd64
chmod 755 yq_linux_amd64
sudo mv yq_linux_amd64 /usr/local/bin/yq

# krew (https://krew.sigs.k8s.io/plugins/)
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)

echo 'export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"' >> ~/.bashrc
. ~/.bashrc

# krew: install a few plugins
kubectl krew install split-yaml
kubectl krew install resource-capacity
kubectl krew install lineage
kubectl krew install rbac-lookup
kubectl krew install access-matrix
kubectl krew install unlimited

# krew examples
kubectl lineage deployments sas-logon-app -n viya4
watch kubectl resource-capacity --pods --util --sort cpu.util -n viya4

# SAS Viya Orders CLI
wget https://github.com/sassoftware/viya4-orders-cli/releases/download/1.6.0/viya4-orders-cli_linux_amd64

sudo mv viya4-orders-cli_linux_amd64 /usr/local/bin/sas-viya-orders
sudo chmod 775 /usr/local/bin/sas-viya-orders

sas-viya-orders -v

# SAS Mirror Manager
wget https://support.sas.com/installation/viya/4/sas-mirror-manager/lax/mirrormgr-linux.tgz
tar xvzf mirrormgr-linux.tgz
rm -f mirrormgr-linux.tgz

sudo mv mirrormgr /usr/local/bin/
rm -rf LICENSE README.md esp-edge-extension/ licenses/

mirrormgr -v

# SAS Viya CLI, download from https://support.sas.com/downloads/package.htm?pid=2512
# Copy CLI package to machine
tar xvzf ~/sas-viya-cli-*-linux-amd64.tgz
rm -f ~/sas-viya-cli-*-linux-amd64.tgz
sudo mv sas-viya /usr/local/bin

sas-viya plugins list-repos
sas-viya plugins list-repo-plugins

sas-viya plugins install --repo sas audit
sas-viya plugins install --repo sas authorization
sas-viya plugins install --repo sas batch
sas-viya plugins install --repo sas cas
sas-viya plugins install --repo sas compute
sas-viya plugins install --repo sas configuration
sas-viya plugins install --repo sas credentials
sas-viya plugins install --repo sas dagentsrv
sas-viya plugins install --repo sas dcmtransfer
sas-viya plugins install --repo sas detection
sas-viya plugins install --repo sas detection-definition
sas-viya plugins install --repo sas detection-message-schema
sas-viya plugins install --repo sas decisiongitdeploy   
sas-viya plugins install --repo sas devices
sas-viya plugins install --repo sas folders
sas-viya plugins install --repo sas fonts
sas-viya plugins install --repo sas identities
sas-viya plugins install --repo sas job
sas-viya plugins install --repo sas launcher
sas-viya plugins install --repo sas licenses
sas-viya plugins install --repo sas listdata
sas-viya plugins install --repo sas mip-migration
sas-viya plugins install --repo sas models
sas-viya plugins install --repo sas notifications
sas-viya plugins install --repo sas oauth
sas-viya plugins install --repo sas reports
sas-viya plugins install --repo sas rfc-solution-config
sas-viya plugins install --repo sas rtdmobjectmigration
sas-viya plugins install --repo sas scoreexecution
sas-viya plugins install --repo sas sid-functions
sas-viya plugins install --repo sas transfer
sas-viya plugins install --repo sas workload-orchestrator
sas-viya plugins install --repo sas visual-forecasting
```



## SAS Viya deployment

``` shell
oc new-project viya4

mkdir -p ~/viya4/site-config/patches
mkdir -p ~/viya4/site-config/resources
mkdir -p ~/viya4/site-config/security/cacerts

tar xvzf ~/SASViyaV4_*.tgz
```



### Apply Viya security constraints

These steps need to be executed by the cluster-admin. 

```shell
cd ~/viya4

# list all available SCCs
find ./sas-bases -name "*scc*.yaml"

# apply most of them
oc create -f ./sas-bases/examples/cas/configure/cas-server-scc.yaml
oc create -f ./sas-bases/examples/configure-elasticsearch/internal/openshift/sas-opendistro-scc.yaml
oc create -f ./sas-bases/examples/restore/scripts/openshift/sas-backup-pv-copy-cleanup-job-scc.yaml
oc create -f ./sas-bases/examples/sas-connect-spawner/openshift/sas-connect-spawner-scc.yaml
oc create -f ./sas-bases/overlays/migration/openshift/migration-job-scc.yaml
oc create -f ./sas-bases/overlays/sas-microanalytic-score/service-account/sas-microanalytic-score-scc.yaml
oc create -f ./sas-bases/overlays/sas-model-repository/service-account/sas-model-repository-scc.yaml

# confirm
oc get scc
```

*Note: repeat this step if the namespace gets re-created*

```shell
oc project viya4

oc -n viya4 adm policy add-scc-to-user sas-cas-server -z sas-cas-server
oc -n viya4 adm policy add-scc-to-user sas-opendistro -z sas-opendistro
oc -n viya4 adm policy add-scc-to-user sas-connect-spawner -z sas-connect-spawner
oc -n viya4 adm policy add-scc-to-user anyuid -z sas-model-publish-kaniko
oc -n viya4 adm policy add-scc-to-user nonroot -z sas-programming-environment
oc -n viya4 adm policy add-scc-to-user sas-microanalytic-score -z sas-microanalytic-score
oc -n viya4 adm policy add-scc-to-user sas-model-repository -z sas-model-repository
```

List bindings to local service accounts

```shell
kubectl get rolebindings,clusterrolebindings --all-namespaces  \
    -o custom-columns='KIND:kind,NAMESPACE:metadata.namespace,NAME:metadata.name,SERVICE_ACCOUNTS:subjects[?(@.kind=="ServiceAccount")].name' \
    | grep "KIND\|viya4"
```



### Patches



#### Patch fsGroup value

Find the fsgroup value and update the fsGroup within the security context of Pods within the cluster. The fsGroup value is a random value and changes when re-creating the project/namespace. Make sure to use the right name for your namespace.

```shell
FSGROUP=$(oc get project viya4 -o jsonpath='{.metadata.annotations.openshift\.io/sa\.scc\.supplemental-groups}' | cut -f1 -d /)
echo $FSGROUP

cp sas-bases/examples/security/container-security/update-fsgroup.yaml site-config/patches/update-fsgroup.yaml
chmod 644 site-config/patches/update-fsgroup.yaml

sed -i "s|{{ FSGROUP_VALUE }}|$FSGROUP|" site-config/patches/update-fsgroup.yaml

# check
head -n 30 site-config/patches/update-fsgroup.yaml
```



#### Storage patches

Used to provide persistent storage to SAS computation engines (MAS, SAS, CAS)

##### MAS

Based on  `$deploy/sas-bases/examples/sas-microanalytic-score/astores`. Provides the MAS engine with persistent storage for saving ASTORE model files.

```yaml
resources:
(...)
- site-config/patches/mas-astore-pvc.yaml
```

##### CAS

(optional) fileshare for CAS for accessing business data. Requires existing PVC.

```yaml
transformers:
- site-config/patches/cas-nfsshare-mount.yaml
```

##### SAS

(optional) fileshare for SAS compute sessions for accessing business data. Requires existing PVC.

```yaml
transformers:
- site-config/patches/sas-nfsshare-mount.yaml
```



#### TLS - store server certificates

TLS options for Routes in OpenShift:

![TLS options in OpenShift](edge_passthrough_reencrypt1.png)

This deployment uses the default TLS certificate stored on the Router ("edge"). This certificate is signed by Let's Encrypt using the following trust chain:

```shell
</dev/null openssl s_client -connect console-openshift-console.apps.rosa.sasviya-5adf.mh2a.p3.openshiftapps.com:443 \
	--showcerts | openssl x509
	
depth=2 C = US, O = Internet Security Research Group, CN = ISRG Root X1
verify return:1
depth=1 C = US, O = Let's Encrypt, CN = R3
verify return:1
depth=0 CN = *.apps.rosa.sasviya-5adf.mh2a.p3.openshiftapps.com
...
```

* C = US, O = Internet Security Research Group, CN = ISRG Root X1
  * C = US, O = Let's Encrypt, CN = R3
    * CN = *.apps.rosa.sasviya-5adf.mh2a.p3.openshiftapps.com

Some components in SAS Viya use a "north-south" communication pattern, so SAS need to trust the LE CA and Root certs. We will add these certificates to the truststores (might not be needed actually - the CA's of the LE certificate could already be known and trusted).

```shell
mkdir -p site-config/security/cacerts

curl -s https://letsencrypt.org/certs/lets-encrypt-r3.pem | openssl x509 -text | \
	sed -n '/-----BEGIN/,/-----END/p' > site-config/security/cacerts/le-r3.pem
curl -s https://letsencrypt.org/certs/isrgrootx1.pem | openssl x509 -text | \
	sed -n '/-----BEGIN/,/-----END/p' > site-config/security/cacerts/le-isrgrootx1.pem

# check
openssl x509 -in site-config/security/cacerts/le-r3.pem -text
openssl x509 -in site-config/security/cacerts/le-isrgrootx1.pem -text
```

The patch to add the new certificates to the truststores:

```shell
# add ca-cert to truststore list
cp sas-bases/examples/security/customer-provided-ca-certificates.yaml site-config/security/
chmod 644 site-config/security/customer-provided-ca-certificates.yaml

sed -i "s|{{ CA_CERTIFICATE_FILE_NAME }}|site-config/security/cacerts/le-isrgrootx1.pem|" site-config/security/customer-provided-ca-certificates.yaml
printf "\n- site-config/security/cacerts/le-r3.pem\n" >> site-config/security/customer-provided-ca-certificates.yaml
```



## Build & deploy

### Simple deploy

```shell
cd ~/viya4
kustomize build -o site.yaml

oc project viya4
kubectl apply --selector="sas.com/admin=cluster-api" --server-side --force-conflicts -f site.yaml
kubectl apply --selector="sas.com/admin=cluster-wide" -f site.yaml
kubectl apply --selector="sas.com/admin=cluster-local" -f site.yaml --prune
kubectl apply --selector="sas.com/admin=namespace" -f site.yaml --prune
```

### Split site.yaml

(optional, useful for submitting selected manifests only)

```shell
cd ~/viya4
rm -rf splityaml/
cat ~/viya4/site.yaml | kubectl split-yaml -p splityaml/
cd splityaml/
```



## Start and Stop

```shell
kubectl create job sas-stop-all-`date +%s` --from cronjobs/sas-stop-all -n viya4
kubectl create job sas-start-all-`date +%s` --from cronjobs/sas-start-all -n viya4
```



