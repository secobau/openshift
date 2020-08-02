In order to install a new Red Hat Openshift cluster in AWS please follow these steps in the precise order:
* https://cloud.redhat.com/openshift/install

All the steps will be performed from an AWS Cloud9 terminal with enough privileges (AdministratorAccess will work).

If you are going to create a cluster in an already existing VPC then you need to create your AWS Cloud9 environment inside a public subnet of your VPC.

Disable AWS managed temporary credentials in AWS Cloud9 settings. Now you need to create a new Access Key in your AWS IAM Security Credentials and then configure your AWS Cloud9 terminal:
```bash
aws configure


```

Now generate an SSH key pair.
```bash
ssh-keygen


```
Choose a version number:
```bash
version=4.4.10


```
Afterwards you can proceed:
```bash
eval "$(ssh-agent -s)"
ssh-add $HOME/.ssh/id_rsa

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
export ClusterName=openshift-internal
export DomainName=sebastian-colomar.com


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
If you wish your cluster to be private and not accessible from the external network:
```bash
sed --in-place s/External/Internal/ $dir/install-config.yaml


```
Be sure to enable the following features in the VPC configuration in case your cluster is private:
```bash
DNS resolution Enabled
DNS hostnames  Enabled


```
In case you want to install your cluster in an already existing VPC then you will need to add the subnet IDs to the platform.aws.subnets field:
```bash
platform:
  aws:
    subnets: 
    - subnet-private111
    - subnet-private222
    - subnet-private333
    
    
```    
In case of using an already existing VPC you will also need to add the CIDR blocks for the machine network which must coincide with the corresponding CIDR blocks for the private subnets:
```bash
networking:
  machineNetwork:
  - cidr: 10.0.1.0/24	
  - cidr: 10.0.3.0/24	
  - cidr: 10.0.5.0/24	


```
It is a good idea to make a copy of your configuration file:
```bash
cp $dir/install-config.yaml $dir/install-config.yaml.$( date +%F_%H%M )


```
Now you can create the cluster in AWS:
```BASH
openshift-install-$version create cluster --dir $dir --log-level=debug


```
If you have correctly created you AWS Cloud9 environment inside a public subnet of the already existing VPC then it should not be the case but if the install process is blocked because there is no DNS resolution of the API URL then you will need to create another Cloud9 environment inside a public subnet in the VPC where your private cluster is being installed.
You will need to import into this new environment the SSH key pair used in the previous environment as well as the folder with the Openshift install files.
You can then start again the install process after removing the terraform.tfstate file.

The installation process will eventually finish successfully.

To access the cluster as the system:admin user when using 'oc', run the following command:
```bash
export KUBECONFIG=$dir/auth/kubeconfig


```
To relax the security in your cluster so that images are not forced to run as a pre-allocated UID, without granting everyone access to the privileged SCC (a better solution is to bind only ephemeral ports in your application):
```bash
oc adm policy add-scc-to-group anyuid system:authenticated


```
