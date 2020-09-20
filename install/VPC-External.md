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
Now you can proceed with the following steps:
* https://github.com/secobau/openshift/blob/master/install/create.md
* https://github.com/secobau/openshift/blob/master/install/certs.md
* https://github.com/secobau/openshift/blob/master/install/scc.md
