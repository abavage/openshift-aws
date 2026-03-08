# OpenShift STS enabled cluster on AWS

## Configuration Steps

### Pre Requisite Steps

* Centos / RHEL Bastion host (x86 or aarch64)
* OpenShift command-line interface [oc](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/)
* OpenShift for x86_64 Installer [openshift-install](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/) [1]
* Cloud Credential Operator CLI utility [ccoctl](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/)
* Valid pull-secret from https://console.redhat.com
* ssh-key pair to access the nodes
* aws-cli installed

[0] All the versions of the utilities above need to match!

[1] - The version of `openshift-install` will dictate the version of OpenShift installed on the cluster. `install-config.yaml` does not expose a version for installation. 

## Installation Pre Requisite

`openshift-install` can either install the cluster as a IPI or UPI cluster. Either let the installer create all the vpc components or build them in advance. `It's strongly recommended to build all componenets in advance (UPI)` then the cluster and vpc componenets are two (2) completly different entities.

* IPI (installer provisioned infrastructure) where the installer configures all the vpc componenets (vpc, subnets, igw, routes, etc). The installer treats the cluster and vpc as a single entity. eg when the cluster is destroyed all vpc components are destroyed.
* UPI (user provisioned infrastructure). The vpc and accociated componets are already provisioned in advance.

### Cluster 
Clusters can eithe be installed as External or Internal facing.
* External - API and Ingress are exposed to the internet via internet facing NLB's.
* Internal - API and Ingress are only accessable to internal clients via internal facing NLB's.

All nodes are installed in private subnets and accessed via NLB's, API and ingress. NLB's are either configured in public subnets, if clusters are external facing or private subnets if clusters are internally facing. 


### VPC Requirments
* VPC sized as 10.0.0.0/16
* DNS resolution enabled
* If Cluster is `Private` - 3 x private subnets one per AZ as /23 or /24.
* If cluster is `Public`, 3 x public subnets one per AZ as /23 or /24 `AND` 3 x private subnets one per AZ as /23 or /24
* Smallest sized subnet would be /27
* Depending on the specific use case managed NAT is a viable option for nodes to egress and reach required registries or a dedicated proxy.
* vpce gateway created and routes added for:
   * s3.`<region>`.amazonaws.com
* vpce created for the following services:
   * sts.`<region>`.amazonaws.com
   * ec2.`<region>`.amazonaws.com
   * elasticloadbalancing.`<region>`.amazonaws.com

### Route53
A single Route53 hosted zone is required to act as the base domain for the cluster. The hosted zone can either be Public or Private. 
* Public: Needs to be a valid IANA registered domain or a delegated subdomian.
* Private: Only accessable to the VPC and componenets within the vpc.

### Bastion Host
* If the cluster is External a laptop or ec2 instance will be sufficient to deploy the cluster
* If the cluster is Internal an ec2 instance in the VPC is needed or a device internally connected to the vpc.
* Centos / RHEL Bastion host (x86 or aarch64)
* AWS credentials as either a user or role/policy with `arn:aws:iam::aws:policy/AdministratorAccess` associated.  
   * Best practice would have a policy with a limited set of permissions to create the cluster. See https://docs.redhat.com.
* All the utilites avaiable see above


## Cluster Installation
When `openshift-install` is run with standard configuration the OpenShift cluster will be installed `without` STS being enabled. The cluster has zero intergation with the AWS infrastructure. The cluster can't be retrofitted after installation to enable STS. Additional steps are required to generate the specific manifests for installation coupled with the Cloud Credential Operator CLI utility to create the Cloud Front origin and s3 bucket for the OICD to work as expetced with the aws STS service. 

### Generate ssh-key
```
$ ssh-keygen -t rsa -N "" -f ~/.ssh/cluster_rsa
```

### pull-secret
```
$ echo "pull-secret" > ~/.config/containers/auth.json
```

### aws command
```
$ aws sts get-caller-identity
```

### Cluster Name
The cluster_name in install-config.yaml `installconfig.spec.metadata.name` and the name used for the `ccoctl --name` option must be identical.

### Create install-config.yaml
```
$ mkdir cluster-install && cd cluster-install
$ mkdir sts-config
$ vim install-config.yaml
$ mkdir installation_dir
$ cp install-config.yaml installation_dir
```

```
# install-config.yaml
apiVersion: v1
additionalTrustBundlePolicy: Proxyonly
baseDomain: apps.example-int.com
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform:
    aws:
      type: m5.xlarge
      zones:
      - ap-southeast-2a
      - ap-southeast-2b
      - ap-southeast-2c
  replicas: 3
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform:
    aws:
      type: m5.xlarge
      zones:
      - ap-southeast-2a
      - ap-southeast-2b
      - ap-southeast-2c
  replicas: 3
metadata:
  name: one
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.0.0.0/16
  serviceNetwork:
  - 172.30.0.0/16
  networkType: OVNKubernetes
platform:
  aws:
    region: ap-southeast-2
    lbType: NLB
    vpc:
      subnets:
      - id: subnet-0dc16b3393c7c4f4c
      - id: subnet-02da6c924d215b397
      - id: subnet-0874ee309e2eb91d3
      - id: subnet-098aa464a25a4b727
      - id: subnet-04eb13a7b562b887b
      - id: subnet-0639cb1ab53f47828
publish: External
credentialsMode: Manual
pullSecret: <pull-secret>
sshKey: <ssh_public_key>

```

### Generate manifests - Part 1
The cluster install version will be `4.20.4`.

```
$ oc adm release extract --credentials-requests --cloud=aws --to=./sts-config quay.io/openshift-release-dev/ocp-release:4.20.4-x86_64

$ oc adm release extract --install-config=./install-config.yaml --included --credentials-requests --cloud=aws --to=./sts-config quay.io/openshift-release-dev/ocp-release:4.20.4-x86_64

```

### Create the CloudFront Origin and s3 Bucket
```
$ ccoctl aws create-all --name=one --region=ap-southeast-2 --output-dir=cco-config --create-private-s3-bucket --credentials-requests-dir=./sts-config

```

### Generate manifests - Part 2
install-config.yaml is consumed/deleted at this stage
```
$ openshift-install create manifests --dir=./installation_dir
$ cp cco-config/manifests/* ./installation_dir/manifests/
$ cp -r cco-config/tls ./installation_dir/
```

### Start the Cluster Build
```
$ openshift-install create cluster --dir=./installation_dir --log-level=info
```

Credentials for the Cluster are displayed at the end of a successful Cluster build.


## Cluster Testing
Items to review

### Login

```
$ oc login <url> -u kubeadmin -p <password>
$ oc get cm cluster-config-v1 -o yaml -n kube-system

# review and confirm the OIDC url is what was created using ccoctl
$ oc get authentication cluster -o yaml
$ oc get cloudcredentials cluster -o yaml
$ oc get secret cloud-credentials -n openshift-ingress-operator -o json | jq -r .data.credentials | base64 -d
$ oc get nodes
$ oc get pods -A
```

## Destroy the Cluster 
```
$ openshift-install destroy cluster --dir=./installation_dir --log-level=info
```

### Destroy the cloudfront oidc / bucket
```
$ ccoctl aws delete --name=one --region=ap-southeast-2
```












