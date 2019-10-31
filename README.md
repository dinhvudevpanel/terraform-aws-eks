# Deploy a full AWS EKS cluster with Terraform

## What resources are created

1. VPC
2. Internet Gateway (IGW)
3. Public and Private Subnets
4. Security Groups, Route Tables and Route Table Associations
5. IAM roles, instance profiles and policies
6. An EKS Cluster
7. Autoscaling group and Launch Configuration
8. Worker Nodes in a private Subnet
9. bastion host for ssh access to the VPC
10. The ConfigMap required to register Nodes with EKS
11. KUBECONFIG file to authenticate kubectl using the `aws eks get-token` command. needs awscli version `1.16.156 >`

## Configuration

You can configure you config with the following input variables:

| Name                      | Description                        | Default                                                                                                                                                                                                                                                                                                                                                                                                          |
| ------------------------- | ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `cluster-name`            | The name of your EKS Cluster       | `eks-cluster`                                                                                                                                                                                                                                                                                                                                                                                                    |
| `aws-region`              | The AWS Region to deploy EKS       | `us-east-2`                                                                                                                                                                                                                                                                                                                                                                                                      |
| `availability-zones`      | AWS Availability Zones             | `["us-east-2a", "us-east-2b", "us-east-2c"]`                                                                                                                                                                                                                                                                                                                                                                     |
| `k8s-version`             | The desired K8s version to launch  | `1.13`                                                                                                                                                                                                                                                                                                                                                                                                           |
| `node-instance-type`      | Worker Node EC2 instance type      | `t2.medium`                                                                                                                                                                                                                                                                                                                                                                                                       |
| `root-block-size`         | Size of the root EBS block device  | `20`                                                                                                                                                                                                                                                                                                                                                                                                             |
| `desired-capacity`        | Autoscaling Desired node capacity  | `2`                                                                                                                                                                                                                                                                                                                                                                                                              |
| `max-size`                | Autoscaling Maximum node capacity  | `5`                                                                                                                                                                                                                                                                                                                                                                                                              |
| `min-size`                | Autoscaling Minimum node capacity  | `1`                                                                                                                                                                                                                                                                                                                                                                                                              |
| `public-min-size`         | Public Node groups ASG capacity    | `1`                                                                                                                                                                                                                                                                                                                                                                                                              |
| `public-max-size`         | Public Node groups ASG capacity    | `1`                                                                                                                                                                                                                                                                                                                                                                                                              |
| `public-desired-capacity` | Public Node groups ASG capacity    | `1`                                                                                                                                                                                                                                                                                                                                                                                                              |
| `vpc-subnet-cidr`         | Subnet CIDR                        | `10.0.0.0/16`                                                                                                                                                                                                                                                                                                                                                                                                    |
| `private-subnet-cidr`     | Private Subnet CIDR                | `["10.0.0.0/19", "10.0.32.0/19", "10.0.64.0/19"]`                                                                                                                                                                                                                                                                                                                                                                |
| `public-subnet-cidr`      | Public Subnet CIDR                 | `["10.0.128.0/20", "10.0.144.0/20", "10.0.160.0/20"]`                                                                                                                                                                                                                                                                                                                                                            |
| `db-subnet-cidr`          | DB/Spare Subnet CIDR               | `["10.0.192.0/21", "10.0.200.0/21", "10.0.208.0/21"]`                                                                                                                                                                                                                                                                                                                                                            |
| `eks-cw-logging`          | EKS Logging Components             | `["api", "audit", "authenticator", "controllerManager", "scheduler"]`                                                                                                                                                                                                                                                                                                                        
### Terraform

You need to run the following commands to create the resources with Terraform:

```bash
terraform init
terraform plan
terraform apply
```


### Setup kubectl

Setup your `KUBECONFIG`

```bash
terraform output kubeconfig > ~/.kube/eks-cluster
export KUBECONFIG=~/.kube/eks-cluster
```

### Authorize worker nodes

Get the config from terraform output, and save it to a yaml file:

```bash
terraform output config-map > config-map-aws-auth.yaml
```

Configure aws cli with a user account having appropriate access and apply the config map to EKS cluster:

```bash
kubectl apply -f config-map-aws-auth.yaml
```

You can verify the worker nodes are joining the cluster

```bash
kubectl get nodes --watch
```

### Authorize users to access the cluster

Initially, only the system that deployed the cluster will be able to access the cluster. To authorize other users for accessing the cluster, `aws-auth` config needs to be modified by using the steps given below:

* Open the aws-auth file in the edit mode on the machine that has been used to deploy EKS cluster:

```bash
sudo kubectl edit -n kube-system configmap/aws-auth
```

* Add the following configuration in that file by changing the placeholders:


```yaml

mapUsers: |
  - userarn: arn:aws:iam::123456789:user/<username>
    username: <username>
    groups:
      - system:masters
```

So, the final configuration would look like this:

```yaml
apiVersion: v1
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789:role/devel-worker-nodes-NodeInstanceRole-74RF4UBDUKL6
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
  mapUsers: |
    - userarn: arn:aws:iam::1234567893:user/<username>
      username: <username>
      groups:
        - system:masters
```

* Once the user map is added in the configuration we need to create cluster role binding for that user:

```bash
kubectl create clusterrolebinding ops-user-cluster-admin-binding-<username> --clusterrole=cluster-admin --user=<username>
```
Replace the placeholder with proper values

### Cleaning up

You can destroy this cluster entirely by running:

```bash
terraform plan -destroy
terraform destroy  --force
```
