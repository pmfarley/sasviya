# SAS Viya on ROSA - deployment notes

[TOC]



## Connect to environment

SSH

```shell
ssh -i "/drives/c/Users/gerhje/OneDrive - SAS/Arbeit/ssh_keys/sasviya-ocp-rosa.id" ec2-user@18.221.95.98
```

OCP Web Console

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

Fileshare (`sas-efs-shared-pvc`) used by SAS, requires `viya4` namespace.

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

