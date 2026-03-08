# OpenShift STS enabled cluster on AWS

## Configuration Steps

### Pre Requisite Steps

* Centos / RHEL Bastion host (x86 or aarch64)
* OpenShift command-line interface [oc](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/)
* OpenShift for x86_64 Installer [openshift-install](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/) [1]
* Cloud Credential Operator CLI utility [ccoctl](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/)
* Valid pull-secret from https://console.redhat.com
* ssh-key pair to access the nodes

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








