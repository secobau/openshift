1. Create a new project:

   `oc new-project quay-enterprise`
1. Install the Red Hat Quay Operator
1. Load credentials to obtain Quay:
   
   ```
   oc get secret -n openshift-config pull-secret -o yaml > registry-pull-secret.yaml
   
   
   ```
   Modify the secret removing all unnecessary metada and load it again:
   ```
   oc apply -n quay-enterprise -f registry-pull-secret.yaml
   
   
   ```
1. Please customize the following configuration before applying it:

   `oc apply -f https://raw.githubusercontent.com/secobau/openshift/master/install/quayEcosystem.yaml`
1. Create the necessary secrets:

   ```
   oc create secret generic registry-admin --from-literal=superuser-username=xxx --from-literal=superuser-password=yyy --from-literal=superuser-email=aaa@bbb.com
   oc create secret generic registry-clair --from-literal=database-username=clair --from-literal=database-password=yyy --from-literal=database-root-password=uuu --from-literal=database-name=clair
   oc create secret generic registry-config --from-literal=config-app-password=xxx
   oc create secret generic registry-quay --from-literal=database-username=quay --from-literal=database-password=yyy --from-literal=database-root-password=uuu --from-literal=database-name=quay
   oc create secret generic registry-redis --from-literal=password=xxx
   oc create secret generic registry-s3 --from-literal=accessKey=xxx --from-literal=secretKey=yyy
   oc create secret tls registry-ssl --key=privkey.pem --cert=fullchain.pem
   
   
   ```
   
