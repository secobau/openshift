First follow these instructions:
* https://github.com/secobau/openshift/blob/master/install/initial.yaml

In case you want to install your cluster in an already existing VPC then you will need to add the subnet IDs to the platform.aws.subnets field:
```bash
platform:
  aws:
    subnets: 
    - subnet-public1111
    - subnet-public2222
    - subnet-public3333
    - subnet-private111
    - subnet-private222
    - subnet-private333
    
    
```    
In case of using an already existing VPC you will also need to add the CIDR block for the machine network which must include the corresponding CIDR blocks for the private and public subnets:
```bash
networking:
  machineNetwork:
  - cidr: 10.0.1.0/24
  - cidr: 10.0.2.0/24
  - cidr: 10.0.3.0/24
  - cidr: 10.0.4.0/24
  - cidr: 10.0.5.0/24
  - cidr: 10.0.6.0/24


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
  oc create configmap custom-ca --from-file=ca-bundle.crt=$dir/tls/chain.pem -n openshift-config


  ```
  3. Update the cluster-wide proxy configuration with the newly created ConfigMap:
  ```bash
  oc patch proxy/cluster --type=merge --patch='{"spec":{"trustedCA":{"name":"custom-ca"}}}'


  ```
  4. Create a secret that contains the wildcard certificate and key:
  ```bash
  oc create secret tls certificate --cert=$dir/tls/cert.pem --key=$dir/tls/privkey.pem -n openshift-ingress


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
  oc create secret tls certificate --cert=$dir/tls/cert.pem --key=$dir/tls/privkey.pem -n openshift-config


  ```
  3. Update the API server to reference the created secret.
  ```bash
  oc patch apiserver cluster --type=merge -p '{"spec":{"servingCerts":{"namedCertificates":[{"names":["api.'$ClusterName'.'$DomainName'"],"servingCertificate":{"name":"certificate"}}]}}}'
  
  
  ```

To relax the security in your cluster so that images are not forced to run as a pre-allocated UID, without granting everyone access to the privileged SCC (a better solution is to bind only ephemeral ports in your application):
```bash
oc adm policy add-scc-to-group anyuid system:authenticated


```
