# ParallelCluster Multi-AZ feature user guide

To fully utilize EC2 instances in a whole Region, you can follow this guide to launch instance across multiple AZs within a single Slurm cluster, both on-demand and spot instances.

*Note 1: Currently, this feature is for Slurm scheduler only.*

*Note 2: Keep in mind cross-AZ traffic cost may incurs if you use it improperly. For example, head node and compute nodes in different AZ and need huge data exchange, access shared storage belongs to another AZ, communications amonge compute nodes across AZ, etc*

## 0. Environment

ParallelCluster Version: 3.1.1

Git repos:

- AWS ParallelCluster CLI: `https://github.com/Chen188/aws-parallelcluster/tree/v3.1.1_maz`
- AWS ParallelCluster Node: `https://github.com/Chen188/aws-parallelcluster-node/tree/v3.1.1_maz`

Package URLs:

- ParallelCluster CLI: `https://github.com/Chen188/aws-parallelcluster/archive/refs/heads/v3.1.1_maz.zip`
- Head node post-install script: `https://s3.ap-northeast-1.amazonaws.com/share.bkt.binc/parallelcluster/3.1.1/custom-actions/on-node-configured-script.sh`

## 1. User guide

### Upgrade ParallelCluster CLI

You’ve to update the cli package(if you happen to have an old version installed), we advise to use a virtual python env, such as `venv` or `conda`. Run the command to upgrade ParallelCluster CLI:

```bash
wget -q https://github.com/Chen188/aws-parallelcluster/archive/refs/heads/v3.1.1_maz.zip
unzip -q v3.1.1_maz.zip && cd aws-parallelcluster-3.1.1_maz
pip install --force-reinstall --upgrade ./cli
```

### Create config file

You can refer the configuration below to activate Multi-AZ feature, note that you can set multiples Subnet IDs under `SlurmQueues/Networking/SubnetIds` , these subnets should belong to different AZs (that’s why we need Multi-AZ).

```yaml
Region: ap-southeast-1
Image:
  Os: ubuntu2004
HeadNode:
  InstanceType: t3.medium
  Networking:
    SubnetId: subnet-0a0baedf6aeb5143b
  Ssh:
    KeyName: new-sing
  CustomActions:
    OnNodeConfigured:
      Script: <paste head node post-install script url>
Scheduling:
  Scheduler: slurm
  SlurmQueues:
  - Name: spot-1
    ComputeResources:
    - Name: g4dn-16xlarge
      InstanceType: g4dn.16xlarge
      MinCount: 0
      MaxCount: 100
    CapacityType: SPOT
    Networking:
      SubnetIds:
      - subnet-xxxxxxx1
      - subnet-xxxxxxx2
  - Name: od-1
    ComputeResources:
    - Name: g4dn-16xlarge
      InstanceType: g4dn.16xlarge
      MinCount: 0
      MaxCount: 100
    Networking:
      SubnetIds:
      - subnet-xxxxxxx1
      - subnet-xxxxxxx2
```

### Create cluster

Run following cmd to create cluster.

`pcluster create-cluster -c your-config-file -n multi-az`

### Enjoy it

Use it as you do before.

## 3. Details about what changed

### AWS ParallelCluster CLI

**Update 1**

Remove the restriction on setting multiple Subnet IDs under Compute Resource. As we mentioned before, now you’re able to set multiple subnets under `Networking/SubnetIds`.

**Update 2**

We’ll put `SubnetIds` config into CloudFormation::Init metadata, and then it’ll be merged into `/etc/chef/dna.json` in Head Node, so that our modified SlurmResume program can know which subnets to use while launching new instances.

**Update 3**

Add `ec2:DescribeLaunchTemplateVersion` permission to HeadNode instance profile, so that SlurmResume program can override the existing launch template’s subnet setting before launching instances.

### AWS ParallelCluster Node

Modify the instance launching logic of SlurmResume. For each Instance launching request, if there are not enough resources in the current Subnet, the next Subnet will be tried in turn, until the resource application is successful or there is no Subnet to try(request failed). For a Subnet has no resource, set a cooldown time of 5 minutes, during which no resource request will be issued to it.

### Custom Action

Used to install customized `aws-parallelcluster-node` package in the Head Node.
