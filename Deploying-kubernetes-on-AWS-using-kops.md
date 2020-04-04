# kops / Deploying kubernetes on AWS using kops

**Table of Contents**
* [Getting Started](#getting-started)
* [Configuration Example](#configuration-example)

## Getting Started

First, complete the ["Getting Started"](https://github.com/kubernetes/kops/blob/master/docs/aws.md) document @ https://github.com/kubernetes/kops/blob/master/docs/aws.md

The [getting started](https://github.com/kubernetes/kops/blob/master/docs/aws.md) tutorial will ask you to install kops, kubectl and setup your AWS environment. The tuturial will require an AWS account where you can create various resources. I've used a personal AWS account in the [configuration example](#configuration-example) and a public domain hosted by route53. There are other domain options including DNS using a .local domain.

**IMPORTANT** make sure you delete your cluster (Step 9) once you are done with this tutorial. You don't want to pay for aws resources that you do not intend to use.

## Configuration example

In the configuration example  I've ommited the installation of kops, kubernetes and setting up your AWS environment. They are covered in the "Getting Started" document and many web sites describe how to handle any issues. The main purpose of this runbook is to show how I used kops to deploy my kubernetes cluster on AWS using the instructions from the "Getting Started" document

**My environment**

- Public DNS (Scenario 1b in ["Getting Started"](https://github.com/kubernetes/kops/blob/master/docs/aws.md)): project4.dev.oyarsa.net
- Region: us-west-2

**Jump to**
- [Step 1 create a project](#step-1-create-a-project)
- [Step 2 create a hosted zone](#step-2-create-a-hosted-zone)
- [Step 3 create bucket to store cluster configs](#step-5-create-bucket-to-store-cluster-configs)
- [Step 4 create cluster config](#step-4-create-cluster-config)
- [Step 5 edit the cluster config if needed](#step-5-edit-the-cluster-config-if-needed)
- [Step 6 start the cluster](#step-6-start-the-cluster)
- [Step 7 validate the cluster has started](#step-7-validate-the-cluster-has-started)
- [Step 8 create a deployment and expose it](#step-8-create-a-deployment-and-expose-it)
- [Step 9 delete cluster](#step-9-delete-cluster)

### Step 1 create a project
```
##################################################################
# Create a working directory
##################################################################
(base) private@ubuntu:~/devops$ mkdir project4; cd project4
(base) private@ubuntu:~/devops/project4$ 

##################################################################
# Set some variables for kops and aws
##################################################################
(base) private@ubuntu:~/devops/project4$ cat project4.rc; source project4.rc 
export EDITOR=/usr/bin/vim
export AWS_DEFAULT_PROFILE=kops
export AWS_DEFAULT_REGION=us-west-2
export NAME=project4.dev.oyarsa.net
export KOPS_STATE_STORE=s3://project4-dev-oyarsa-net-state-store
```

### Step 2 create a hosted zone
```
##################################################################
# Create zone. Save nameservers output for subdomain.json file 
# described later. 
##################################################################
(base) private@ubuntu:~/devops/project4$ ID=$(uuidgen) && aws route53 create-hosted-zone --name project4.dev.oyarsa.net --caller-reference $ID | \
>     jq .DelegationSet.NameServers
[
  "ns-49.awsdns-06.com",
  "ns-1343.awsdns-39.org",
  "ns-934.awsdns-52.net",
  "ns-1771.awsdns-29.co.uk"
]

##################################################################
# Get hosted zone ID. We'll use this later.
##################################################################
(base) private@ubuntu:~/devops/project4$ aws route53 list-hosted-zones | jq '.HostedZones[] | select(.Name=="oyarsa.net.") | .Id'
"/hostedzone/Z19KYSK4809JXK"

##################################################################
# Create subdomain input file. Update the domain name and 
# nameservers from the previous output.
##################################################################
(base) private@ubuntu:~/devops/project4$ cat subdomain.json 
{
  "Comment": "Create a subdomain NS record in the parent domain",
  "Changes": [
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "project4.dev.oyarsa.net",
        "Type": "NS",
        "TTL": 300,
        "ResourceRecords": [
          {
            "Value": "ns-49.awsdns-06.com"
          },
          {
            "Value": "ns-1343.awsdns-39.org"
          },
          {
            "Value": "ns-934.awsdns-52.net"
          },
          {
            "Value": "ns-1771.awsdns-29.co.uk"
          }
        ]
      }
    }
  ]
}

##################################################################
# Create a subdomain NS record in the parent domain
##################################################################
(base) private@ubuntu:~/devops/project4$ aws route53 change-resource-record-sets \
>  --hosted-zone-id Z19KYSK4809JXK \
>  --change-batch file://subdomain.json
{
    "ChangeInfo": {
        "Id": "/change/C3KI0E77CVLQX4",
        "Status": "PENDING",
        "SubmittedAt": "2019-06-18T02:32:36.187Z",
        "Comment": "Create a subdomain NS record in the parent domain"
    }
}

##################################################################
# Describe hosted zone to verify
##################################################################
(base) private@ubuntu:~/devops/project4$ aws route53 get-hosted-zone --id Z2RE54LR5QLE2R
{
    "HostedZone": {
        "Id": "/hostedzone/Z2RE54LR5QLE2R",
        "Name": "project4.dev.oyarsa.net.",
        "CallerReference": "cd127e4e-15cb-49ac-8639-31f046abfbb4",
        "Config": {
            "PrivateZone": false
        },
        "ResourceRecordSetCount": 2
    },
    "DelegationSet": {
        "NameServers": [
            "ns-49.awsdns-06.com",
            "ns-1343.awsdns-39.org",
            "ns-934.awsdns-52.net",
            "ns-1771.awsdns-29.co.uk"
        ]
    }
}

##################################################################
# Lookup nameservers to validate
##################################################################
(base) private@ubuntu:~/devops/project4$ dig NS project4.dev.oyarsa.net +short
ns-1343.awsdns-39.org.
ns-1771.awsdns-29.co.uk.
ns-49.awsdns-06.com.
ns-934.awsdns-52.net.
```

### Step 3 create bucket to store cluster configs
```
(base) private@ubuntu:~/devops/project4$ aws s3api create-bucket \
>     --bucket project4-dev-oyarsa-net-state-store \
>     --region us-east-1
{
    "Location": "/project4-dev-oyarsa-net-state-store"
}
```
### Step 4 create cluster config
```
(base) private@ubuntu:~/devops/project4$ kops create cluster --zones us-west-2a ${NAME}
I0617 19:39:47.818141   98613 create_cluster.go:519] Inferred --cloud=aws from zone "us-west-2a"
I0617 19:39:48.005262   98613 subnets.go:184] Assigned CIDR 172.20.32.0/19 to subnet us-west-2a
I0617 19:39:49.937294   98613 create_cluster.go:1486] Using SSH public key: /home/private/.ssh/id_rsa.pub
Previewing changes that will be made:


*********************************************************************************

A new kops version is available: 1.12.1

Upgrading is recommended
More information: https://github.com/kubernetes/kops/blob/master/permalinks/upgrade_kops.md#1.12.1

*********************************************************************************

I0617 19:39:51.606516   98613 executor.go:103] Tasks: 0 done / 85 total; 43 can run
I0617 19:39:52.277637   98613 executor.go:103] Tasks: 43 done / 85 total; 24 can run
I0617 19:39:52.843975   98613 executor.go:103] Tasks: 67 done / 85 total; 16 can run
I0617 19:39:53.120330   98613 executor.go:103] Tasks: 83 done / 85 total; 2 can run
I0617 19:39:53.233955   98613 executor.go:103] Tasks: 85 done / 85 total; 0 can run
Will create resources:
  AutoscalingGroup/master-us-west-2a.masters.project4.dev.oyarsa.net
  	Granularity         	1Minute
  	LaunchConfiguration 	name:master-us-west-2a.masters.project4.dev.oyarsa.net
  	MaxSize             	1
  	Metrics             	[GroupDesiredCapacity, GroupInServiceInstances, GroupMaxSize, GroupMinSize, GroupPendingInstances, GroupStandbyInstances, GroupTerminatingInstances, GroupTotalInstances]
  	MinSize             	1
  	Subnets             	[name:us-west-2a.project4.dev.oyarsa.net]
  	SuspendProcesses    	[]
  	Tags                	{Name: master-us-west-2a.masters.project4.dev.oyarsa.net, KubernetesCluster: project4.dev.oyarsa.net, k8s.io/cluster-autoscaler/node-template/label/kops.k8s.io/instancegroup: master-us-west-2a, k8s.io/role/master: 1}

  AutoscalingGroup/nodes.project4.dev.oyarsa.net
  	Granularity         	1Minute
  	LaunchConfiguration 	name:nodes.project4.dev.oyarsa.net
  	MaxSize             	2
  	Metrics             	[GroupDesiredCapacity, GroupInServiceInstances, GroupMaxSize, GroupMinSize, GroupPendingInstances, GroupStandbyInstances, GroupTerminatingInstances, GroupTotalInstances]
  	MinSize             	2
  	Subnets             	[name:us-west-2a.project4.dev.oyarsa.net]
  	SuspendProcesses    	[]
  	Tags                	{k8s.io/cluster-autoscaler/node-template/label/kops.k8s.io/instancegroup: nodes, k8s.io/role/node: 1, Name: nodes.project4.dev.oyarsa.net, KubernetesCluster: project4.dev.oyarsa.net}

  DHCPOptions/project4.dev.oyarsa.net
  	DomainName          	us-west-2.compute.internal
  	DomainNameServers   	AmazonProvidedDNS
  	Shared              	false
  	Tags                	{Name: project4.dev.oyarsa.net, KubernetesCluster: project4.dev.oyarsa.net, kubernetes.io/cluster/project4.dev.oyarsa.net: owned}

  EBSVolume/a.etcd-events.project4.dev.oyarsa.net
  	AvailabilityZone    	us-west-2a
  	Encrypted           	false
  	SizeGB              	20
  	Tags                	{Name: a.etcd-events.project4.dev.oyarsa.net, KubernetesCluster: project4.dev.oyarsa.net, k8s.io/etcd/events: a/a, k8s.io/role/master: 1, kubernetes.io/cluster/project4.dev.oyarsa.net: owned}
  	VolumeType          	gp2

  EBSVolume/a.etcd-main.project4.dev.oyarsa.net
  	AvailabilityZone    	us-west-2a
  	Encrypted           	false
  	SizeGB              	20
  	Tags                	{k8s.io/etcd/main: a/a, k8s.io/role/master: 1, kubernetes.io/cluster/project4.dev.oyarsa.net: owned, Name: a.etcd-main.project4.dev.oyarsa.net, KubernetesCluster: project4.dev.oyarsa.net}
  	VolumeType          	gp2

  IAMInstanceProfile/masters.project4.dev.oyarsa.net
  	Shared              	false

  IAMInstanceProfile/nodes.project4.dev.oyarsa.net
  	Shared              	false

  IAMInstanceProfileRole/masters.project4.dev.oyarsa.net
  	InstanceProfile     	name:masters.project4.dev.oyarsa.net id:masters.project4.dev.oyarsa.net
  	Role                	name:masters.project4.dev.oyarsa.net

  IAMInstanceProfileRole/nodes.project4.dev.oyarsa.net
  	InstanceProfile     	name:nodes.project4.dev.oyarsa.net id:nodes.project4.dev.oyarsa.net
  	Role                	name:nodes.project4.dev.oyarsa.net

  IAMRole/masters.project4.dev.oyarsa.net
  	ExportWithID        	masters

  IAMRole/nodes.project4.dev.oyarsa.net
  	ExportWithID        	nodes

  IAMRolePolicy/masters.project4.dev.oyarsa.net
  	Role                	name:masters.project4.dev.oyarsa.net

  IAMRolePolicy/nodes.project4.dev.oyarsa.net
  	Role                	name:nodes.project4.dev.oyarsa.net

  InternetGateway/project4.dev.oyarsa.net
  	VPC                 	name:project4.dev.oyarsa.net
  	Shared              	false
  	Tags                	{Name: project4.dev.oyarsa.net, KubernetesCluster: project4.dev.oyarsa.net, kubernetes.io/cluster/project4.dev.oyarsa.net: owned}

  Keypair/apiserver-aggregator
  	Signer              	name:apiserver-aggregator-ca id:cn=apiserver-aggregator-ca
  	Subject             	cn=aggregator
  	Type                	client
  	Format              	v1alpha2

  Keypair/apiserver-aggregator-ca
  	Subject             	cn=apiserver-aggregator-ca
  	Type                	ca
  	Format              	v1alpha2

  Keypair/apiserver-proxy-client
  	Signer              	name:ca id:cn=kubernetes
  	Subject             	cn=apiserver-proxy-client
  	Type                	client
  	Format              	v1alpha2

  Keypair/ca
  	Subject             	cn=kubernetes
  	Type                	ca
  	Format              	v1alpha2

  Keypair/etcd-clients-ca
  	Subject             	cn=etcd-clients-ca
  	Type                	ca
  	Format              	v1alpha2

  Keypair/etcd-manager-ca-events
  	Subject             	cn=etcd-manager-ca-events
  	Type                	ca
  	Format              	v1alpha2

  Keypair/etcd-manager-ca-main
  	Subject             	cn=etcd-manager-ca-main
  	Type                	ca
  	Format              	v1alpha2

  Keypair/etcd-peers-ca-events
  	Subject             	cn=etcd-peers-ca-events
  	Type                	ca
  	Format              	v1alpha2

  Keypair/etcd-peers-ca-main
  	Subject             	cn=etcd-peers-ca-main
  	Type                	ca
  	Format              	v1alpha2

  Keypair/kops
  	Signer              	name:ca id:cn=kubernetes
  	Subject             	o=system:masters,cn=kops
  	Type                	client
  	Format              	v1alpha2

  Keypair/kube-controller-manager
  	Signer              	name:ca id:cn=kubernetes
  	Subject             	cn=system:kube-controller-manager
  	Type                	client
  	Format              	v1alpha2

  Keypair/kube-proxy
  	Signer              	name:ca id:cn=kubernetes
  	Subject             	cn=system:kube-proxy
  	Type                	client
  	Format              	v1alpha2

  Keypair/kube-scheduler
  	Signer              	name:ca id:cn=kubernetes
  	Subject             	cn=system:kube-scheduler
  	Type                	client
  	Format              	v1alpha2

  Keypair/kubecfg
  	Signer              	name:ca id:cn=kubernetes
  	Subject             	o=system:masters,cn=kubecfg
  	Type                	client
  	Format              	v1alpha2

  Keypair/kubelet
  	Signer              	name:ca id:cn=kubernetes
  	Subject             	o=system:nodes,cn=kubelet
  	Type                	client
  	Format              	v1alpha2

  Keypair/kubelet-api
  	Signer              	name:ca id:cn=kubernetes
  	Subject             	cn=kubelet-api
  	Type                	client
  	Format              	v1alpha2

  Keypair/master
  	AlternateNames      	[100.64.0.1, 127.0.0.1, api.internal.project4.dev.oyarsa.net, api.project4.dev.oyarsa.net, kubernetes, kubernetes.default, kubernetes.default.svc, kubernetes.default.svc.cluster.local]
  	Signer              	name:ca id:cn=kubernetes
  	Subject             	cn=kubernetes-master
  	Type                	server
  	Format              	v1alpha2

  LaunchConfiguration/master-us-west-2a.masters.project4.dev.oyarsa.net
  	AssociatePublicIP   	true
  	IAMInstanceProfile  	name:masters.project4.dev.oyarsa.net id:masters.project4.dev.oyarsa.net
  	ImageID             	kope.io/k8s-1.12-debian-stretch-amd64-hvm-ebs-2019-05-13
  	InstanceType        	m3.medium
  	RootVolumeSize      	64
  	RootVolumeType      	gp2
  	SSHKey              	name:kubernetes.project4.dev.oyarsa.net-f5:e5:9f:b9:89:b2:fc:b3:d1:37:ad:2b:79:de:e6:00 id:kubernetes.project4.dev.oyarsa.net-f5:e5:9f:b9:89:b2:fc:b3:d1:37:ad:2b:79:de:e6:00
  	SecurityGroups      	[name:masters.project4.dev.oyarsa.net]
  	SpotPrice           	

  LaunchConfiguration/nodes.project4.dev.oyarsa.net
  	AssociatePublicIP   	true
  	IAMInstanceProfile  	name:nodes.project4.dev.oyarsa.net id:nodes.project4.dev.oyarsa.net
  	ImageID             	kope.io/k8s-1.12-debian-stretch-amd64-hvm-ebs-2019-05-13
  	InstanceType        	t2.medium
  	RootVolumeSize      	128
  	RootVolumeType      	gp2
  	SSHKey              	name:kubernetes.project4.dev.oyarsa.net-f5:e5:9f:b9:89:b2:fc:b3:d1:37:ad:2b:79:de:e6:00 id:kubernetes.project4.dev.oyarsa.net-f5:e5:9f:b9:89:b2:fc:b3:d1:37:ad:2b:79:de:e6:00
  	SecurityGroups      	[name:nodes.project4.dev.oyarsa.net]
  	SpotPrice           	

  ManagedFile/etcd-cluster-spec-events
  	Location            	backups/etcd/events/control/etcd-cluster-spec

  ManagedFile/etcd-cluster-spec-main
  	Location            	backups/etcd/main/control/etcd-cluster-spec

  ManagedFile/manifests-etcdmanager-events
  	Location            	manifests/etcd/events.yaml

  ManagedFile/manifests-etcdmanager-main
  	Location            	manifests/etcd/main.yaml

  ManagedFile/project4.dev.oyarsa.net-addons-bootstrap
  	Location            	addons/bootstrap-channel.yaml

  ManagedFile/project4.dev.oyarsa.net-addons-core.addons.k8s.io
  	Location            	addons/core.addons.k8s.io/v1.4.0.yaml

  ManagedFile/project4.dev.oyarsa.net-addons-dns-controller.addons.k8s.io-k8s-1.12
  	Location            	addons/dns-controller.addons.k8s.io/k8s-1.12.yaml

  ManagedFile/project4.dev.oyarsa.net-addons-dns-controller.addons.k8s.io-k8s-1.6
  	Location            	addons/dns-controller.addons.k8s.io/k8s-1.6.yaml

  ManagedFile/project4.dev.oyarsa.net-addons-dns-controller.addons.k8s.io-pre-k8s-1.6
  	Location            	addons/dns-controller.addons.k8s.io/pre-k8s-1.6.yaml

  ManagedFile/project4.dev.oyarsa.net-addons-kube-dns.addons.k8s.io-k8s-1.12
  	Location            	addons/kube-dns.addons.k8s.io/k8s-1.12.yaml

  ManagedFile/project4.dev.oyarsa.net-addons-kube-dns.addons.k8s.io-k8s-1.6
  	Location            	addons/kube-dns.addons.k8s.io/k8s-1.6.yaml

  ManagedFile/project4.dev.oyarsa.net-addons-kube-dns.addons.k8s.io-pre-k8s-1.6
  	Location            	addons/kube-dns.addons.k8s.io/pre-k8s-1.6.yaml

  ManagedFile/project4.dev.oyarsa.net-addons-kubelet-api.rbac.addons.k8s.io-k8s-1.9
  	Location            	addons/kubelet-api.rbac.addons.k8s.io/k8s-1.9.yaml

  ManagedFile/project4.dev.oyarsa.net-addons-limit-range.addons.k8s.io
  	Location            	addons/limit-range.addons.k8s.io/v1.5.0.yaml

  ManagedFile/project4.dev.oyarsa.net-addons-rbac.addons.k8s.io-k8s-1.8
  	Location            	addons/rbac.addons.k8s.io/k8s-1.8.yaml

  ManagedFile/project4.dev.oyarsa.net-addons-storage-aws.addons.k8s.io-v1.6.0
  	Location            	addons/storage-aws.addons.k8s.io/v1.6.0.yaml

  ManagedFile/project4.dev.oyarsa.net-addons-storage-aws.addons.k8s.io-v1.7.0
  	Location            	addons/storage-aws.addons.k8s.io/v1.7.0.yaml

  Route/0.0.0.0/0
  	RouteTable          	name:project4.dev.oyarsa.net
  	CIDR                	0.0.0.0/0
  	InternetGateway     	name:project4.dev.oyarsa.net

  RouteTable/project4.dev.oyarsa.net
  	VPC                 	name:project4.dev.oyarsa.net
  	Shared              	false
  	Tags                	{KubernetesCluster: project4.dev.oyarsa.net, kubernetes.io/cluster/project4.dev.oyarsa.net: owned, kubernetes.io/kops/role: public, Name: project4.dev.oyarsa.net}

  RouteTableAssociation/us-west-2a.project4.dev.oyarsa.net
  	RouteTable          	name:project4.dev.oyarsa.net
  	Subnet              	name:us-west-2a.project4.dev.oyarsa.net

  SSHKey/kubernetes.project4.dev.oyarsa.net-f5:e5:9f:b9:89:b2:fc:b3:d1:37:ad:2b:79:de:e6:00
  	KeyFingerprint      	42:91:20:31:1a:ce:40:6c:c9:f0:10:3e:ab:c6:8e:6c

  Secret/admin

  Secret/kube

  Secret/kube-proxy

  Secret/kubelet

  Secret/system:controller_manager

  Secret/system:dns

  Secret/system:logging

  Secret/system:monitoring

  Secret/system:scheduler

  SecurityGroup/masters.project4.dev.oyarsa.net
  	Description         	Security group for masters
  	VPC                 	name:project4.dev.oyarsa.net
  	RemoveExtraRules    	[port=22, port=443, port=2380, port=2381, port=4001, port=4002, port=4789, port=179]
  	Tags                	{Name: masters.project4.dev.oyarsa.net, KubernetesCluster: project4.dev.oyarsa.net, kubernetes.io/cluster/project4.dev.oyarsa.net: owned}

  SecurityGroup/nodes.project4.dev.oyarsa.net
  	Description         	Security group for nodes
  	VPC                 	name:project4.dev.oyarsa.net
  	RemoveExtraRules    	[port=22]
  	Tags                	{Name: nodes.project4.dev.oyarsa.net, KubernetesCluster: project4.dev.oyarsa.net, kubernetes.io/cluster/project4.dev.oyarsa.net: owned}

  SecurityGroupRule/all-master-to-master
  	SecurityGroup       	name:masters.project4.dev.oyarsa.net
  	SourceGroup         	name:masters.project4.dev.oyarsa.net

  SecurityGroupRule/all-master-to-node
  	SecurityGroup       	name:nodes.project4.dev.oyarsa.net
  	SourceGroup         	name:masters.project4.dev.oyarsa.net

  SecurityGroupRule/all-node-to-node
  	SecurityGroup       	name:nodes.project4.dev.oyarsa.net
  	SourceGroup         	name:nodes.project4.dev.oyarsa.net

  SecurityGroupRule/https-external-to-master-0.0.0.0/0
  	SecurityGroup       	name:masters.project4.dev.oyarsa.net
  	CIDR                	0.0.0.0/0
  	Protocol            	tcp
  	FromPort            	443
  	ToPort              	443

  SecurityGroupRule/master-egress
  	SecurityGroup       	name:masters.project4.dev.oyarsa.net
  	CIDR                	0.0.0.0/0
  	Egress              	true

  SecurityGroupRule/node-egress
  	SecurityGroup       	name:nodes.project4.dev.oyarsa.net
  	CIDR                	0.0.0.0/0
  	Egress              	true

  SecurityGroupRule/node-to-master-tcp-1-2379
  	SecurityGroup       	name:masters.project4.dev.oyarsa.net
  	Protocol            	tcp
  	FromPort            	1
  	ToPort              	2379
  	SourceGroup         	name:nodes.project4.dev.oyarsa.net

  SecurityGroupRule/node-to-master-tcp-2382-4000
  	SecurityGroup       	name:masters.project4.dev.oyarsa.net
  	Protocol            	tcp
  	FromPort            	2382
  	ToPort              	4000
  	SourceGroup         	name:nodes.project4.dev.oyarsa.net

  SecurityGroupRule/node-to-master-tcp-4003-65535
  	SecurityGroup       	name:masters.project4.dev.oyarsa.net
  	Protocol            	tcp
  	FromPort            	4003
  	ToPort              	65535
  	SourceGroup         	name:nodes.project4.dev.oyarsa.net

  SecurityGroupRule/node-to-master-udp-1-65535
  	SecurityGroup       	name:masters.project4.dev.oyarsa.net
  	Protocol            	udp
  	FromPort            	1
  	ToPort              	65535
  	SourceGroup         	name:nodes.project4.dev.oyarsa.net

  SecurityGroupRule/ssh-external-to-master-0.0.0.0/0
  	SecurityGroup       	name:masters.project4.dev.oyarsa.net
  	CIDR                	0.0.0.0/0
  	Protocol            	tcp
  	FromPort            	22
  	ToPort              	22

  SecurityGroupRule/ssh-external-to-node-0.0.0.0/0
  	SecurityGroup       	name:nodes.project4.dev.oyarsa.net
  	CIDR                	0.0.0.0/0
  	Protocol            	tcp
  	FromPort            	22
  	ToPort              	22

  Subnet/us-west-2a.project4.dev.oyarsa.net
  	ShortName           	us-west-2a
  	VPC                 	name:project4.dev.oyarsa.net
  	AvailabilityZone    	us-west-2a
  	CIDR                	172.20.32.0/19
  	Shared              	false
  	Tags                	{Name: us-west-2a.project4.dev.oyarsa.net, KubernetesCluster: project4.dev.oyarsa.net, kubernetes.io/cluster/project4.dev.oyarsa.net: owned, SubnetType: Public, kubernetes.io/role/elb: 1}

  VPC/project4.dev.oyarsa.net
  	CIDR                	172.20.0.0/16
  	EnableDNSHostnames  	true
  	EnableDNSSupport    	true
  	Shared              	false
  	Tags                	{Name: project4.dev.oyarsa.net, KubernetesCluster: project4.dev.oyarsa.net, kubernetes.io/cluster/project4.dev.oyarsa.net: owned}

  VPCDHCPOptionsAssociation/project4.dev.oyarsa.net
  	VPC                 	name:project4.dev.oyarsa.net
  	DHCPOptions         	name:project4.dev.oyarsa.net

Must specify --yes to apply changes

Cluster configuration has been created.

Suggestions:
 * list clusters with: kops get cluster
 * edit this cluster with: kops edit cluster project4.dev.oyarsa.net
 * edit your node instance group: kops edit ig --name=project4.dev.oyarsa.net nodes
 * edit your master instance group: kops edit ig --name=project4.dev.oyarsa.net master-us-west-2a

Finally configure your cluster with: kops update cluster --name project4.dev.oyarsa.net --yes
```

### Step 5 edit the cluster config if needed
```
(base) private@ubuntu:~/devops/project4$ kops edit cluster ${NAME}
Edit cancelled, no changes made.
```

### Step 6 start the cluster
```
(base) private@ubuntu:~/devops/project4$ kops update cluster ${NAME} --yes

*********************************************************************************

A new kops version is available: 1.12.1

Upgrading is recommended
More information: https://github.com/kubernetes/kops/blob/master/permalinks/upgrade_kops.md#1.12.1

*********************************************************************************

I0618 00:13:16.955753   47611 executor.go:103] Tasks: 0 done / 85 total; 43 can run
I0618 00:13:21.137547   47611 vfs_castore.go:729] Issuing new certificate: "apiserver-aggregator-ca"
I0618 00:13:21.746067   47611 vfs_castore.go:729] Issuing new certificate: "etcd-clients-ca"
I0618 00:13:22.330442   47611 vfs_castore.go:729] Issuing new certificate: "etcd-peers-ca-events"
I0618 00:13:22.641309   47611 vfs_castore.go:729] Issuing new certificate: "etcd-peers-ca-main"
I0618 00:13:22.746970   47611 vfs_castore.go:729] Issuing new certificate: "etcd-manager-ca-events"
I0618 00:13:23.718856   47611 vfs_castore.go:729] Issuing new certificate: "ca"
I0618 00:13:23.770154   47611 vfs_castore.go:729] Issuing new certificate: "etcd-manager-ca-main"
I0618 00:13:25.019173   47611 executor.go:103] Tasks: 43 done / 85 total; 24 can run
I0618 00:13:28.316493   47611 vfs_castore.go:729] Issuing new certificate: "kubecfg"
I0618 00:13:28.671584   47611 vfs_castore.go:729] Issuing new certificate: "apiserver-proxy-client"
I0618 00:13:30.018092   47611 vfs_castore.go:729] Issuing new certificate: "apiserver-aggregator"
I0618 00:13:30.228618   47611 vfs_castore.go:729] Issuing new certificate: "kubelet"
I0618 00:13:30.692400   47611 vfs_castore.go:729] Issuing new certificate: "kube-proxy"
I0618 00:13:31.097725   47611 vfs_castore.go:729] Issuing new certificate: "master"
I0618 00:13:31.315639   47611 vfs_castore.go:729] Issuing new certificate: "kubelet-api"
I0618 00:13:31.438481   47611 vfs_castore.go:729] Issuing new certificate: "kube-controller-manager"
I0618 00:13:32.641504   47611 vfs_castore.go:729] Issuing new certificate: "kops"
I0618 00:13:32.986479   47611 vfs_castore.go:729] Issuing new certificate: "kube-scheduler"
I0618 00:13:34.402608   47611 executor.go:103] Tasks: 67 done / 85 total; 16 can run
I0618 00:13:35.647736   47611 executor.go:103] Tasks: 83 done / 85 total; 2 can run
I0618 00:13:36.403123   47611 executor.go:103] Tasks: 85 done / 85 total; 0 can run
I0618 00:13:36.403211   47611 dns.go:153] Pre-creating DNS records
I0618 00:13:37.637852   47611 update_cluster.go:291] Exporting kubecfg for cluster
kops has set your kubectl context to project4.dev.oyarsa.net

Cluster is starting.  It should be ready in a few minutes.

Suggestions:
 * validate cluster: kops validate cluster
 * list nodes: kubectl get nodes --show-labels
 * ssh to the master: ssh -i ~/.ssh/id_rsa admin@api.project4.dev.oyarsa.net
 * the admin user is specific to Debian. If not using Debian please use the appropriate user based on your OS.
 * read about installing addons at: https://github.com/kubernetes/kops/blob/master/docs/addons.md.
```

### Step 7 validate the cluster has started
```
##################################################################
# Validate cluster. It may take serveral minutes for the cluster to start.
##################################################################
(base) private@ubuntu:~/devops/project4$ kops validate cluster
Using cluster from kubectl context: project4.dev.oyarsa.net

Validating cluster project4.dev.oyarsa.net

INSTANCE GROUPS
NAME			ROLE	MACHINETYPE	MIN	MAX	SUBNETS
master-us-west-2a	Master	m3.medium	1	1	us-west-2a
nodes			Node	t2.medium	2	2	us-west-2a

NODE STATUS
NAME						ROLE	READY
ip-172-20-46-124.us-west-2.compute.internal	master	True
ip-172-20-62-58.us-west-2.compute.internal	node	True
ip-172-20-63-144.us-west-2.compute.internal	node	True

Your cluster project4.dev.oyarsa.net is ready

##################################################################
# Get cluster info
##################################################################
(base) private@ubuntu:~/devops/project4$ kops get clusters project4.dev.oyarsa.net
NAME			CLOUD	ZONES
project4.dev.oyarsa.net	aws	us-west-2a

##################################################################
# Get k8s pods
##################################################################
(base) private@ubuntu:~/devops/project4$ kubectl -n kube-system get po
NAME                                                                  READY   STATUS    RESTARTS   AGE
dns-controller-7cdbd4d448-b2qvf                                       1/1     Running   0          5m56s
etcd-manager-events-ip-172-20-46-124.us-west-2.compute.internal       1/1     Running   0          4m56s
etcd-manager-main-ip-172-20-46-124.us-west-2.compute.internal         1/1     Running   0          5m43s
kube-apiserver-ip-172-20-46-124.us-west-2.compute.internal            1/1     Running   2          5m24s
kube-controller-manager-ip-172-20-46-124.us-west-2.compute.internal   1/1     Running   0          5m11s
kube-dns-57dd96bb49-d58sq                                             3/3     Running   0          4m17s
kube-dns-57dd96bb49-vkl2p                                             3/3     Running   1          5m56s
kube-dns-autoscaler-867b9fd49d-lvnhx                                  1/1     Running   2          5m55s
kube-proxy-ip-172-20-46-124.us-west-2.compute.internal                1/1     Running   0          5m41s
kube-proxy-ip-172-20-62-58.us-west-2.compute.internal                 1/1     Running   0          4m11s
kube-proxy-ip-172-20-63-144.us-west-2.compute.internal                1/1     Running   0          4m22s
kube-scheduler-ip-172-20-46-124.us-west-2.compute.internal            1/1     Running   0          5m24s

##################################################################
# Get k8s nodes
##################################################################
(base) private@ubuntu:~/devops/project4$ kubectl get nodes
NAME                                          STATUS   ROLES    AGE     VERSION
ip-172-20-49-66.us-west-2.compute.internal    Ready    master   3m57s   v1.12.8
ip-172-20-51-209.us-west-2.compute.internal   Ready    node     3m2s    v1.12.8
ip-172-20-62-153.us-west-2.compute.internal   Ready    node     2m53s   v1.12.8
```

### Step 8 create a deployment and expose it
```
##################################################################
# Run a simple hello-world deployment
##################################################################
(base) private@ubuntu:~/devops/project4$ kubectl run hello-world-project4 --replicas=3 --labels="run=load-balancer-example-project4" \
>   --image=gcr.io/google-samples/node-hello:1.0 --port=8080
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/hello-world-project4 created

##################################################################
# Get deployments
##################################################################
(base) private@ubuntu:~/devops/project4$ kubectl get deployments
NAME                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
hello-world-project4   3         3         3            3           39s

##################################################################
# Get service
##################################################################
(base) private@ubuntu:~/devops/project4$ kubectl get service
NAME                  TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)          AGE
kubernetes            ClusterIP      100.64.0.1       <none>                                                                    443/TCP          8m23s
my-service-project4   LoadBalancer   100.65.142.117   a99351e3d919a11e9bcaa0696bd434c3-1696082251.us-west-2.elb.amazonaws.com   8080:31621/TCP   31s

##################################################################
# Last, test your external service using the ELB aws name
##################################################################
(base) private@ubuntu:~/devops/project4$ curl http://a99351e3d919a11e9bcaa0696bd434c3-1696082251.us-west-2.elb.amazonaws.com:8080
Hello Kubernetes!
```

### Step 9 delete cluster
```
(base) private@ubuntu:~/devops/project4$ kops delete cluster --name ${NAME} --yes
```
