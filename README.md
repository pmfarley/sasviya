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

#### TLS - store server certificates

We won't store a local TLS certificate in the namespace, but rather use the default one (HAProxy). Note: we might want to still store the CA certificates for north-south communication (although it's probably not needed in this case - the LE certificate CAs should already be known).

```shell
mkdir -p site-config/security/cacerts

# copy CA certificate to site-config
cp ~/certs/DigiCertCA.crt site-config/security/cacerts/DigiCert.pem

# check
openssl x509 -in site-config/security/ingress-server-cert.pem -text

# add ca-cert to truststore list
cp sas-bases/examples/security/customer-provided-ca-certificates.yaml site-config/security/
chmod 644 site-config/security/customer-provided-ca-certificates.yaml

sed -i "s|{{ CA_CERTIFICATE_FILE_NAME }}|site-config/security/cacerts/ingress-server-cacerts.pem|" site-config/security/customer-provided-ca-certificates.yaml

echo "- site-config/security/cacerts/DigiCert.pem" >> site-config/security/customer-provided-ca-certificates.yaml
```

