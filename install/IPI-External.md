In order to install a new Red Hat Openshift cluster in AWS please follow these steps in the precise order.

You will need the Pull Secret generated in this page:
* https://cloud.redhat.com/openshift/install

All the steps will be performed from an AWS Cloud9 terminal with enough privileges (AdministratorAccess will work).

You will need to obtain a valid public domain name before installing the cluster:
* https://console.aws.amazon.com/route53/home

Disable AWS managed temporary credentials in AWS Cloud9 settings. Now you need to create a new Access Key in your AWS IAM Security Credentials and then configure your AWS Cloud9 terminal:
```bash
aws configure


```

Now generate an SSH key pair and add it to the SSH agent. This will allow you to access the cluster nodes through SSH.
```bash
ssh-keygen

eval "$(ssh-agent -s)"
ssh-add $HOME/.ssh/id_rsa


```
Choose a version number:
```bash
version=4.5.5


```
Afterwards you can proceed to install the client and the installer binaries:
```bash
for mode in client install
do
  wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/$version/openshift-$mode-linux-$version.tar.gz
  gunzip openshift-$mode-linux-$version.tar.gz
  tar xf openshift-$mode-linux-$version.tar
  rm openshift-$mode-linux-$version.tar
done
mkdir --parents $HOME/bin
for binary in kubectl oc
do
  mv $binary $HOME/bin
done
mv openshift-install $HOME/bin/openshift-install-$version


```
Now you introduce your choice for the name and domain of the cluster:
```bash
export ClusterName=openshift
export DomainName=sebastian-colomar.es


```
Now you create a configuration file template to be later modified:
```bash
dir="$HOME/environment/openshift/install/$ClusterName.$DomainName"
test -d $dir || mkdir --parents $dir
openshift-install-$version create install-config --dir $dir


```
The following script will modify the EC2 instance type so as to choose the cheapest possible type but big enough to correctly set up the cluster:
```bash
cd $dir
wget https://raw.githubusercontent.com/secobau/openshift/master/install/fix-config.sh
chmod +x fix-config.sh && ./fix-config.sh


```
It is a good idea to make a copy of your configuration file:
```bash
cp $dir/install-config.yaml $dir/install-config.yaml.$( date +%F_%H%M )


```
Now you can create the cluster in AWS:
```BASH
openshift-install-$version create cluster --dir $dir --log-level debug


```
The installation process will eventually finish successfully.

To access the cluster as the system:admin user when using 'oc', run the following command:
```bash
export KUBECONFIG=$dir/auth/kubeconfig


```
In order to substitute the self-signed certificate by a valid one:
* https://docs.openshift.com/container-platform/4.4/authentication/certificates/replacing-default-ingress-certificate.html
  
  1. If you need to generate LetsEncrypt certificates you can run this script:
  ```bash
  export EmailAddress=sebastian.colomar@gmail.com
  docker run -it --rm -v ~/.aws/credentials:/root/.aws/credentials -v ~/environment/certs:/etc/letsencrypt certbot/dns-route53 certonly -n --dns-route53 --agree-tos --email $EmailAddress -d *.apps.$ClusterName.$DomainName
  docker run -it --rm -v ~/.aws/credentials:/root/.aws/credentials -v ~/environment/certs:/etc/letsencrypt certbot/dns-route53 certificates
  sudo chown $USER. -R ~/environment/certs
  cp ~/environment/certs/live/apps.$ClusterName.$DomainName/*.pem ~/environment/openshift/install/$ClusterName.$DomainName/tls/


  ```
  2. Create a ConfigMap that includes the certificate authority used to sign the new certificate:
  ```bash
  oc create configmap custom-ca --from-file=ca-bundle.crt=$dir/tls/fullchain.pem -n openshift-config


  ```
  3. Update the cluster-wide proxy configuration with the newly created ConfigMap:
  ```bash
  oc patch proxy/cluster --type=merge --patch='{"spec":{"trustedCA":{"name":"custom-ca"}}}'


  ```
  4. Create a secret that contains the wildcard certificate and key:
  ```bash
  oc create secret tls certificate --cert=$dir/tls/fullchain.pem --key=$dir/tls/privkey.pem -n openshift-ingress


  ```
  5. Update the Ingress Controller configuration with the newly created secret:
  ```bash
  oc patch ingresscontroller.operator default --type=merge -p '{"spec":{"defaultCertificate": {"name": "certificate"}}}' -n openshift-ingress-operator


  ```

* https://docs.openshift.com/container-platform/4.4/authentication/certificates/api-server.html

  1. If you need to generate LetsEncrypt certificates you can run this script:
  ```bash
  export EmailAddress=sebastian.colomar@gmail.com
  docker run -it --rm -v ~/.aws/credentials:/root/.aws/credentials -v ~/environment/certs:/etc/letsencrypt certbot/dns-route53 certonly -n --dns-route53 --agree-tos --email $EmailAddress -d api.$ClusterName.$DomainName
  docker run -it --rm -v ~/.aws/credentials:/root/.aws/credentials -v ~/environment/certs:/etc/letsencrypt certbot/dns-route53 certificates
  sudo chown $USER. -R ~/environment/certs
  cp ~/environment/certs/live/api.$ClusterName.$DomainName/*.pem ~/environment/openshift/install/$ClusterName.$DomainName/tls/


  ```
  2. Create a secret that contains the certificate and key in the openshift-config namespace.
  ```bash
  oc create secret tls certificate --cert=$dir/tls/fullchain.pem --key=$dir/tls/privkey.pem -n openshift-config


  ```
  3. Update the API server to reference the created secret.
  ```bash
  oc patch apiserver cluster --type=merge -p '{"spec":{"servingCerts":{"namedCertificates":[{"names":["api.'$ClusterName'.'$DomainName'"],"servingCertificate":{"name":"certificate"}}]}}}'
  
  
  ```

To relax the security in your cluster so that images are not forced to run as a pre-allocated UID, without granting everyone access to the privileged SCC (a better solution is to bind only ephemeral ports in your application):
```bash
oc adm policy add-scc-to-group anyuid system:authenticated


```
