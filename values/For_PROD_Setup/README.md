Please download the  python script to merge values.yaml files with best effort to preserve comments, formatting, 
and order of items from 

https://github.com/Aref-Riant/yaml-merger-py

Requirements:
```text
pip install ruamel.yaml
pip install mergedeep
```

usage:
```
python yaml-merger.py file1.yaml file2.yaml > mergedfile.yaml
```
---
After you download this got repo do:
```text
cd values/For_PROD_Setup
```
Instead of installing Artifactory, Xray and JAS all in one shot, it is recommended to :
a) first install  Artifactory , login to it and set the base url
b) install Xray and verifuy it successfully connects to the Artifactory instance
c) Do Xray DB Sync
d) Then enable JAS

The steps to do the above are explained in this Readme. 
You can also review the blog - [A Guide to Installing the JFrog Platform on Amazon EKS](https://jfrog.com/blog/install-artifactory-on-eks/) 
, that outlines the  prerequisites and steps required to install and configure the JFrog Platform in Amazon EKS, 
including setting up two AWS systems: IAM Roles for Service Accounts (IRSA) and Application Load Balancer (ALB).

Set the following Environmental variables based on your Deployment K8s environment where you will install the 
JFrog Platform.

Note: the CLOUD_PROVIDER can be gcp or aws ( JFrog Helm charts support Azure as well but this readme was created 
only based on gcp or aws  )

**Environment variables:**
```text
export CLOUD_PROVIDER=gcp
export MY_NAMESPACE=ps-jfrog-platform
export MY_HELM_RELEASE=ps-jfrog-platform-release

export MASTER_KEY=$(openssl rand -hex 32)
# Save this master key to reuse it later
echo ${MASTER_KEY}
# or you can hardcode it to
export MASTER_KEY=02ba23e285e065d2a372b889ac3dbd51510dd0399875f95294312634f50b6960

export JOIN_KEY=$(openssl rand -hex 32)
# Save this join key to reuse it later
echo ${JOIN_KEY}
# or you can hardcode it to
export JOIN_KEY=763d4bdf02ff4cc16d7c5cf2abeccf3f243b5557bf738ec5438fd55df0cec3cc

export ADMIN_USERNAME=admin
export ADMIN_PASSWORD=Test@123

export DB_SERVER=100.185.45.104


export RT_DATABASE_USER=artifactory
export RT_DATABASE_PASSWORD=password
export ARTIFACTORY_DB=sivas-helm-ha-db

export MY_RABBITMQ_ADMIN_USER_PASSWORD=Test@123
export XRAY_DATABASE_USER=xray
export XRAY_DATABASE_PASSWORD=password
export XRAY_DB=xray
```
---

Prepare the K8s environment:
Note: These commands will be useful if you want to iterate run the helm release multiple times i.e  you are not 
starting with a clean k8s environment
```text
helm uninstall $MY_HELM_RELEASE --namespace
or to rollback a revision:
helm rollback $MY_HELM_RELEASE REVISION_NUMBER -n $MY_NAMESPACE
```


To get the release name of a Helm chart, you can use the following command:
```text
helm list -n  $MY_NAMESPACE

NAME                     	NAMESPACE        	REVISION	UPDATED                             	STATUS  	CHART                 	APP VERSION
ps-jfrog-platform-release	ps-jfrog-platform	3       	2023-07-10 12:33:19.393492 -0700 PDT	deployed	jfrog-platform-10.13.1	7.59.9
```

Replace <namespace> with the actual namespace where the Helm release is deployed. 
If you don't specify the --namespace flag, it will list releases across all namespaces.


Delete PVCs as needed:
```text
kubectl delete pvc artifactory-volume-$MY_HELM_RELEASE-artifactory-0 -n $MY_NAMESPACE
kubectl delete pvc data-$MY_HELM_RELEASE-rabbitmq-0 -n $MY_NAMESPACE
kubectl delete pvc data-volume-$MY_HELM_RELEASE-xray-0 -n $MY_NAMESPACE
etc
```

Delete Namespace only if needed as this will delete all the secrets as well:
```text
kubectl delete ns  $MY_NAMESPACE
```

---

If you are starting with a clean k8s environment:

Create Namespace:
```
kubectl create ns  $MY_NAMESPACE
```

---
**Create the secrets**

**Master and Join Keys:**
```text
kubectl delete secret generic masterkey-secret  -n $MY_NAMESPACE
kubectl delete secret generic joinkey-secret   -n $MY_NAMESPACE

kubectl create secret generic masterkey-secret --from-literal=master-key=${MASTER_KEY} -n $MY_NAMESPACE
kubectl create secret generic joinkey-secret --from-literal=join-key=${JOIN_KEY} -n $MY_NAMESPACE
```

**License:**
Create a secret for license with the dataKey as "artifactory.lic" for HA or standalone ( if you want you can name the 
dataKey as artifactory.cluster.license for HA but not necessary) :
```text
kubectl delete secret  artifactory-license  -n $MY_NAMESPACE

kubectl create secret generic artifactory-license --from-file=artifactory.lic=/Users/sureshv/Documents/Test_Scripts/helm_upgrade/licenses/art.lic -n $MY_NAMESPACE

kubectl get secret artifactory-license -o yaml -n $MY_NAMESPACE

kubectl get secret artifactory-license -o json -n $MY_NAMESPACE | jq -r '.data."artifactory.lic"' | base64 --decode

```


Note: if you create it as the following then the dataKey will be art.lic ( i.e same as the name of the file)
```text
kubectl create secret generic artifactory-license \  
--from-file=/Users/sureshv/Documents/Test_Scripts/helm_upgrade/licenses/art.lic -n $MY_NAMESPACE
```



---

Now construct the value.yaml you will finally use:

Ref: https://github.com/jfrog/charts/blob/master/stable/artifactory/values-large.yaml
File mentioned below are in [For_PROD_Setup](values/For_PROD_Setup)
1. Start with the 1_artifactory-values-small.yaml for TEST environment or 1_artifactory-values-large.yaml for PROD 
   environment

```text
python yaml-merger.py 0_values-dynata-artifactory-xray-platform_prod_$CLOUD_PROVIDER.yaml 1_artifactory-values-small.yaml > tmp/1_mergedfile.yaml
or
python yaml-merger.py 0_values-dynata-artifactory-xray-platform_prod_$CLOUD_PROVIDER.yaml 1_artifactory-values-large.yaml > tmp/1_mergedfile.yaml
```

---

2. Override using the 2_artifactory_db_passwords.yaml
   **Artifactory Database Credentials:**
```text
kubectl delete secret  artifactory-database-creds  -n $MY_NAMESPACE

kubectl create secret generic artifactory-database-creds \
--from-literal=db-user=$RT_DATABASE_USER \
--from-literal=db-password=$RT_DATABASE_PASSWORD \
--from-literal=db-url=jdbc:postgresql://$DB_SERVER:5432/$ARTIFACTORY_DB -n $MY_NAMESPACE
```

```
python yaml-merger.py tmp/1_mergedfile.yaml 2_artifactory_db_passwords.yaml > tmp/2_mergedfile.yaml
```
---

3. Override using 3_artifactory_admin_user.yaml 

**The artifactory default admin user secret:**
https://jfrog.com/help/r/artifactory-how-to-unlock-a-user-s-who-is-locked-out-of-artifactory-and-recover-admin-account/non-admin-user-recovery

```text
kubectl delete secret  art-creds  -n $MY_NAMESPACE

kubectl create secret generic art-creds --from-literal=bootstrap.creds='admin@*=Test@123' -n $MY_NAMESPACE
```

```
python yaml-merger.py tmp/2_mergedfile.yaml 3_artifactory_admin_user.yaml > tmp/3_mergedfile.yaml
```
---

4. Override the binaryStore
With AWS https://jfrog.com/help/r/jfrog-installation-setup-documentation/s3-direct-upload-template-recommended :
```
kubectl apply -f 4_custom-binarystore-s3-direct-use_instance-creds.yaml -n $MY_NAMESPACE
```
or

With GCP ( google-storage-v2-direct template configuration from https://jfrog.com/help/r/jfrog-installation-setup-documentation/google-storage-binary-provider-native-client-template ):
```
kubectl delete secret  artifactory-gcp-creds -n $MY_NAMESPACE

kubectl create secret generic artifactory-gcp-creds --from-file=/Users/sureshv/.gcp/gcp.credentials.json \
-n $MY_NAMESPACE

envsubst < binarystore_config/custom-binarystore-gcp.tmpl > binarystore_config/custom-binarystore.yaml

kubectl apply -f binarystore_config/custom-binarystore.yaml -n $MY_NAMESPACE
```
---

5.  Override the system.yaml using either 5_artifactory_system_small.yaml for TEST environment or 
    5_artifactory_system_large.yaml for PROD
See  https://jfrog.com/help/r/how-do-i-tune-artifactory-for-heavy-loads/how-do-i-tune-artifactory-for-heavy-loads
 
```
kubectl delete secret artifactory-custom-systemyaml -n $MY_NAMESPACE
kubectl create secret generic artifactory-custom-systemyaml --from-file=system.yaml=./5_artifactory_system_small.yaml \
-n $MY_NAMESPACE
or
kubectl create secret generic artifactory-custom-systemyaml --from-file=system.yaml=./5_artifactory_system_large.yaml \
-n $MY_NAMESPACE
```
---

6. Override with the 6_xray_db_passwords_pod_size-values-small.yaml for TEST environment or 
   6_xray_db_passwords_pod_size-values-large.yaml for PROD

```text
kubectl delete secret generic xray-database-creds -n $MY_NAMESPACE
kubectl create secret generic xray-database-creds \
--from-literal=db-user=$XRAY_DATABASE_USER \
--from-literal=db-password=$XRAY_DATABASE_PASSWORD \
--from-literal=db-url=postgres://$DB_SERVER:5432/$XRAY_DB\?sslmode=disable -n $MY_NAMESPACE

If "jq --version" >=1.6  where jq  @base64d filter is avaiable use :

kubectl get secret xray-database-creds  -n $MY_NAMESPACE -o json | jq '.data | map_values(@base64d)'

otherwise use:
bash decode_secret.sh <secret-to-decrypt>  <namespace>
```

```
python yaml-merger.py tmp/3_mergedfile.yaml 6_xray_db_passwords_pod_size-values-small.yaml > tmp/6_mergedfile.yaml
or
python yaml-merger.py tmp/3_mergedfile.yaml 6_xray_db_passwords_pod_size-values-large.yaml > tmp/6_mergedfile.yaml
```

7.Rabbitmq

Search "memoryHighWatermark" and found new setting "vm_memory_high_watermark_absolute" that is not in
https://github.com/jfrog/charts/blob/master/stable/xray/values-large.yaml
REf: "vm_memory_high_watermark_absolute" is  a construct from 
https://jfrog.slack.com/archives/CD30SKMDG/p1678277533753349 that was picked from the rabbitmq bitnami chart 
https://github.com/bitnami/charts/blob/main/bitnami/rabbitmq/values.yaml#L476

Ref to maxAvailableSchedulers and onlineSchedulers is in 
https://github.com/bitnami/charts/blob/5492c138a533177ebf1dc660ad19eb18b96f39ba/bitnami/rabbitmq/values.yaml#L210


Rabbitmq is anyway external but maintained by JFrog Platform chart :
From [#249001](https://groups.google.com/a/jfrog.com/g/support-followup/c/STPhVtUGzW4/m/nzIPInHOAAAJ)
You can pass the rabbitmq username , password , url as a secret by creating a secret as below:
```
kubectl delete secret xray-rabbitmq-creds -n $MY_NAMESPACE

kubectl create secret generic xray-rabbitmq-creds --from-literal=username=admin \
--from-literal=password=$MY_RABBITMQ_ADMIN_USER_PASSWORD \
--from-literal=url=amqp://$MY_HELM_RELEASE-rabbitmq:5672 -n $MY_NAMESPACE

kubectl get secret xray-rabbitmq-creds  -n $MY_NAMESPACE -o json | jq '.data | map_values(@base64d)'

```
First get the default load_definition.json  ( from your earlier deploys before you maje the load_definition as secret):
```
kubectl get secrets | grep load-definition -n $MY_NAMESPACE
kubectl get secret <secret-name> -n <namespace> -o json | jq -r '.data["key"]' | base64 --decode
kubectl get secret $MY_HELM_RELEASE-load-definition -n $MY_NAMESPACE -o json | jq -r '.data["load_definition.json"]' 
| base64 --decode

```
and then make load_definition also as a secret after changing the admin password in it:
```
kubectl delete secret $MY_HELM_RELEASE-load-definition -n $MY_NAMESPACE

kubectl create secret generic $MY_HELM_RELEASE-load-definition \
--from-file=load_definition.json=./10_optional_load_definition.json -n $MY_NAMESPACE

kubectl get secret $MY_HELM_RELEASE-load-definition -n $MY_NAMESPACE -o json | jq '.data | map_values(@base64d)'
```

Note: If you already deployed rabbitmq from a previous Xray install you can get the  pod yal definition using:
```text
kubectl get pod $MY_HELM_RELEASE-rabbitmq-0 -n $MY_NAMESPACE -o yaml > ps-jfrog-platform-release-rabbitmq-0.yaml
```

We  want to override the rabbitmq admin user to admin ( instead of guest) and
password (default is password) :
https://github.com/jfrog/charts/blob/b8a04c8f57f7b87d1895cd455fa4859de5db9db2/stable/xray/values.yaml#L484:
Ticket 256917 .
```text
rabbitmq:
  auth:
    username: guest
    password: password
```

by setting the rabbitMQ admin credentials by setting the **existingPasswordSecret** in the helm values.yaml as
mentioned in the comment in snippet below ?
https://jfrog.slack.com/archives/CD30SKMDG/p1686040418572439?thread_ts=1683027138.354849&cid=CD30SKMDG
~~~
rabbitmq:
  enabled: true
  ## Enable the flag if the feature flags in rabbitmq is enabled manually
  rabbitmqUpgradeReady: false
  replicaCount: 1
  rbac:
    create: true
  image:
    registry: releases-docker.jfrog.io
    repository: bitnami/rabbitmq
    tag: 3.11.10-debian-11-r5
  auth:
    username: guest
    password: password
    ## Alternatively, you can use a pre-existing secret with a key called rabbitmq-password by specifying existingPasswordSecret
    # existingPasswordSecret: <name-of-existing-secret>
~~~~
To do this :
a) **Create the rabbitmq-admin-creds:**
```text
kubectl delete secret rabbitmq-admin-creds -n $MY_NAMESPACE 

kubectl create secret generic rabbitmq-admin-creds \
--from-literal=rabbitmq-password=$MY_RABBITMQ_ADMIN_USER_PASSWORD -n $MY_NAMESPACE 

kubectl get secret rabbitmq-admin-creds -n $MY_NAMESPACE -o json | jq '.data | map_values(@base64d)'
kubectl get secret  jfrog-platform-rabbitmq -n devops-acc-us-env -o json | jq '.data | map_values(@base64d)'
```

b) Override with the 7_rabbitmq_enabled_external_values-small.yaml ( for TEST) or 
7_rabbitmq_enabled_external_values-large.yaml ( for PROD) to use the rabbitmq-admin-creds to set the rabbitmq admin 
password.

Note: Even in PROD currently you need to use replicaCount = 1 for the rabbitmq pod because
Rabbitmq in HA mode is not fully supported by Xray product and we have  open JIRA
[XRAY-16820](https://jfrog-int.atlassian.net/browse/XRAY-16820) . See https://jfrog.slack.com/archives/CD30SKMDG/p1688621345420649?thread_ts=1688614562.639429&cid=CD30SKMDG
```
python yaml-merger.py tmp/6_mergedfile.yaml 7_rabbitmq_enabled_external_values-small.yaml > tmp/7_mergedfile.yaml
or
python yaml-merger.py tmp/6_mergedfile.yaml 7_rabbitmq_enabled_external_values-large.yaml > tmp/7_mergedfile.yaml
```
---

Now start deploying the helm release to install the JFrog Products starting with Artifactory:

First do a  Dry run:
```text
helm  upgrade --install $MY_HELM_RELEASE \
-f tmp/3_mergedfile.yaml \
--namespace $MY_NAMESPACE jfrog/jfrog-platform  \
--set gaUpgradeReady=true \
--dry-run
```

---

Then apply without --dry-run :

Make sure you created all the k8s secrets mentioned above . Then make the necessary changes in 3_mergedfile.yaml and 
you can Install  Artifactory HA  with say replicaCount=2 .
```text
helm  upgrade --install $MY_HELM_RELEASE \
-f tmp/3_mergedfile.yaml \
--namespace $MY_NAMESPACE jfrog/jfrog-platform  \
--set gaUpgradeReady=true
```
---

Check Artifactory logs to verify that it can connect to the filestore and database and can start successfully :
```text
kubectl exec -it $MY_HELM_RELEASE-artifactory-0 -n $MY_NAMESPACE -c artifactory -- bash
cd /opt/jfrog/artifactory/var/log
tail -F /opt/jfrog/artifactory/var/log/artifactory-service.log

or

kubectl exec -it $MY_HELM_RELEASE-artifactory-0 -n $MY_NAMESPACE -c artifactory -- tail -F /opt/jfrog/artifactory/var/log/artifactory-service.log
```
---

Get the nginx external IP/url using:
```
kubectl get svc $MY_HELM_RELEASE-artifactory-nginx -n $MY_NAMESPACE
```
For me it was 104.196.98.19 .

---

Next install xray . 
If you have ALB use that instead of nginx in "--set global.jfrogUrlUI" in below command
```text
helm  upgrade --install $MY_HELM_RELEASE \
-f tmp/7_mergedfile.yaml \
--namespace $MY_NAMESPACE jfrog/jfrog-platform  \
--set gaUpgradeReady=true \
--set global.jfrogUrlUI="http://104.196.98.19" 
```

---
If Xray is not connecting to Rabbitmq and you see   following errors in the xray logs:
```
2023-07-23 18:58:25.284035+00:00 [error] <0.28993.9> PLAIN login refused: user 'admin' - invalid credentials
```
It means setting the "xray-rabbitmq-creds" in Values.rabbitmq.external.secrets is not overriding the Rabbitmq admin 
password in the xray system.yaml though Rabbitmq is started successfully with the correct  admin credentials from secret rabbitmq-admin-creds
as shown above. 

Overriding the "-set rabbitmq.external.password" may not work because I am already using rabbitmq.external.secrets 
in the values.yaml ( for the rabbitmq and xray)
In  https://github.com/jfrog/charts/blob/4ba461c93ece4b736db84954982cf4e7ec54f8eb/stable/xray/values.yaml#L189-L193 we can use the "password: "{{ .Values.rabbitmq.external.password }}""  i.e from "-set rabbitmq.external.password" only if Values.rabbitmq.external.secrets is not used.
```
      {{- if not .Values.rabbitmq.external.secrets }}
        url: "{{ tpl .Values.rabbitmq.external.url . }}"
        username: "{{ .Values.rabbitmq.external.username }}"
        password: "{{ .Values.rabbitmq.external.password }}"
      {{- end }}
```
I posted this to  https://jfrog.slack.com/archives/CD30SKMDG/p1690306078218199 and logged INST-6705

To **workaround** this I exported the xray system.yaml  to [8_xray_system_yaml.yaml](8_xray_system_yaml.yaml).
Set the shared.rabbitMq.password to using the correct "clear_text_admin_password_for_rabbitmq" .
Then create the secret:

```
kubectl delete secret xray-custom-systemyaml -n $MY_NAMESPACE
kubectl create secret generic xray-custom-systemyaml --from-file=system.yaml=./8_xray_system_yaml.yaml \
-n $MY_NAMESPACE

```
Then use secret xray-custom-systemyaml to do the systemYamlOverride for xray: 
```
python yaml-merger.py tmp/7_mergedfile.yaml 8_override_xray_system_yaml_in_values.yaml > tmp/8_mergedfile.yaml
``` 
Next upgrade install to do the systemYamlOverride for xray . If you have ALB use that instead of nginx
in "--set global.jfrogUrlUI" in below command
```text
helm  upgrade --install $MY_HELM_RELEASE \
-f tmp/8_mergedfile.yaml \
--namespace $MY_NAMESPACE jfrog/jfrog-platform  \
--set gaUpgradeReady=true \
--set global.jfrogUrlUI="http://104.196.98.19" 
```

Verify using:
```text
$kubectl logs $MY_HELM_RELEASE-rabbitmq-0 -n $MY_NAMESPACE

kubectl exec -it $MY_HELM_RELEASE-rabbitmq-0  -n $MY_NAMESPACE -- bash
find / -name rabbitmq.conf
cat /opt/bitnami/rabbitmq/etc/rabbitmq/rabbitmq.conf

rabbitmqctl status
rabbitmqctl cluster_status
rabbitmqctl list_queues

SSH to the rabbitmq pod and run below curl command:
Note: the default admin password for rabbitMQ is password but we did override it to Test@123 as mentioned above:

curl --user admin:Test@123 http://localhost:15672/api/vhosts
curl --user admin:Test@123 "http://jfrog-platform-rabbitmq:15672"
```

SSH to the Xray server
```text
kubectl exec -it $MY_HELM_RELEASE-xray-0 -n $MY_NAMESPACE -c xray-server -- bash
Example:
kubectl exec -it $MY_HELM_RELEASE-xray-0 -n $MY_NAMESPACE -c xray-server -- bash

cd /opt/jfrog/xray/var/etc
cat /opt/jfrog/xray/var/etc/system.yaml


cd /opt/jfrog/xray/var/log
cat /opt/jfrog/xray/var/log/xray-server-service.log
tail -F /opt/jfrog/xray/var/log/xray-server-service.log
```
---
**Enable JAS**

If xray is up and is now integrated with Artifactory , next  enable JAS in the values.yaml:

```
python yaml-merger.py tmp/8_mergedfile.yaml 9_enable_JAS.yaml > tmp/9_mergedfile.yaml
```

Next do the helm upgrade to install / enable JAS:
```text
helm  upgrade --install $MY_HELM_RELEASE \
-f tmp/9_mergedfile.yaml \
--namespace $MY_NAMESPACE jfrog/jfrog-platform  \
--set gaUpgradeReady=true \
--set global.jfrogUrlUI="http://104.196.98.19" \
--dry-run
```

---

Note: JAS runs as a k8s job , so you will see the pods from the job only when you "Scan for Contextual Analysis".
At that time when you run the following ti will show the pods that are running for the job.
```text
watch kubectl get pods  -n $MY_NAMESPACE
```


----

