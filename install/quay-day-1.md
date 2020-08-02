1. Create a new project:

   `oc new-project quay-enterprise`
1. Install the Red Hat Quay Operator
1. Load credentials to obtain Quay:
   
   ```
   oc get secret -n openshift-config pull-secret -o yaml > pull-secret.yaml
   
   
   ```
   Modify the secret removing all unnecessary metada and load it again:
   ```
   oc apply -n quay-enterprise -f pull-secret.yaml
   
   
   ```
1. Please customize the following configuration before applying it:

   `oc apply -f https://raw.githubusercontent.com/secobau/openshift/master/install/quayEcosystem.yaml`
1. Create the necessary secrets:

   ```
   oc create secret tls custom-quay-ssl --key=privkey.pem --cert=cert.pem
   oc create secret generic quay-admin --from-literal=superuser-username=xxx --from-literal=superuser-password=yyy --from-literal=superuser-email=aaa@bbb.com
   oc create secret generic quay-config-app --from-literal=config-app-password=xxx
   oc create secret generic quay-database-credential --from-literal=database-username=xxx --from-literal=database-password=yyy --from-literal=database-root-password=uuu --from-literal=database-name=vvv
   oc create secret generic redis-password --from-literal=password=xxx
   oc create secret generic s3-credentials --from-literal=accessKey=xxx --from-literal=secretKey=yyy
   
   
   ```
   
