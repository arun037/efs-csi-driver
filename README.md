# efs-cs-driver
Installing efs-csi-driver in eks cluster

Step 1:- Go to https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html

Step 2:- scroll a little bit ,under the pre-requisite Click on the Create an IAM OIDC provider for your cluster 

Routed to https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html

Step 3:- cluster_name=<eks cluster name>

Step 4:- oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)

Step 5:- echo $oidc_id

Step 6:- aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4

Step 7:- eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve

Once done with the above steps, go back to the main documentation page (Step 1) and need to follow the steps

Step 8:- export cluster_name=my-cluster
export role_name=AmazonEKS_EFS_CSI_DriverRole
eksctl create iamserviceaccount \
    --name efs-csi-controller-sa \
    --namespace kube-system \
    --cluster $cluster_name \
    --role-name $role_name \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEFSCSIDriverPolicy \
    --approve
TRUST_POLICY=$(aws iam get-role --role-name $role_name --query 'Role.AssumeRolePolicyDocument' | \
    sed -e 's/efs-csi-controller-sa/efs-csi-*/' -e 's/StringEquals/StringLike/')
aws iam update-assume-role-policy --role-name $role_name --policy-document "$TRUST_POLICY"

Step 9:- Go to the link https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/docs/iam-policy-create.md

Copy the efs-service-account.yaml and change the role-arn which can be found in the role(AmazonEKS_EFS_CSI_DriverRole) created in iam in aws
annotations:
    eks.amazonaws.com/role-arn: <arn:aws:iam::111122223333:role/EKS_EFS_CSI_DriverRole>

    vi efs-service-account.yaml && k create -f efs-service-account.yaml && k get sa -n kube-system


Step 10:- Under the "Get the Amazon EFS CSI driver", self-managed installation of the Amazon EFS CSI driver link has been given . Need to click on the link ,It will route to https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/docs/README.md#installation

Need to follow the steps under [ Manifest (private registry) ]

Step 11:- kubectl kustomize \
    "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=master" > private-ecr-driver.yaml
    (Changed release-2.X to master, since it is not working when we use release)

Step 12:- sed -i.bak -e 's|us-west-2|region-code|' private-ecr-driver.yaml  (Need to change the region)

Step 13:- sed -i.bak -e 's|602401143452|account|' private-ecr-driver.yaml  (Need to change the account# which can be found on the right top corner of the aws ui)

Step 14:- Since we have already created the service account on the step 9, open the private-ecr-driver.yaml which was downloaded on the step 11, need to remove the first yaml manifest for efs-csi-controller-sa

  this needs to be deleted.

  Step 15:- kubectl apply -f private-ecr-driver.yaml

  Step 16:- Go to main documentation file (step 1)
  Go to Create an Amazon EFS file system, under link has been provided to create efs file system

  https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/docs/efs-create-filesystem.md

  and follow the steps

 (change the name of the cluster)

  vpc_id=$(aws eks describe-cluster \
    --name my-cluster \
    --query "cluster.resourcesVpcConfig.vpcId" \
    --output text)
------------------------------------------------------
(Change the region)

cidr_range=$(aws ec2 describe-vpcs \
    --vpc-ids $vpc_id \
    --query "Vpcs[].CidrBlock" \
    --output text \
    --region region-code)
-------------------------------------------------------------
(Change the group name and description as per your wish)

security_group_id=$(aws ec2 create-security-group \
    --group-name MyEfsSecurityGroup \
    --description "My EFS security group" \
    --vpc-id $vpc_id \
    --output text)
---------------------------------------------------------------
aws ec2 authorize-security-group-ingress \
    --group-id $security_group_id \
    --protocol tcp \
    --port 2049 \
    --cidr $cidr_range
----------------------------------------------------------------
(Change the region)

file_system_id=$(aws efs create-file-system \
    --region region-code \
    --performance-mode generalPurpose \
    --query 'FileSystemId' \
    --output text)
--------------------------------------------------------------
kubectl get nodes
---------------------------------------------------------------
aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$vpc_id" \
    --query 'Subnets[*].{SubnetId: SubnetId,AvailabilityZone: AvailabilityZone,CidrBlock: CidrBlock}' \
    --output table

Output will show like this 

---------------------------------------------------------------------
|                          DescribeSubnets                          |
+------------------+-------------------+----------------------------+
| AvailabilityZone |     CidrBlock     |         SubnetId           |
+------------------+-------------------+----------------------------+
|  us-east-1a      |  192.168.96.0/19  |  subnet-0f589bc69b5cd355b  |
|  us-east-1a      |  192.168.32.0/19  |  subnet-0bf72a7d1671407a3  |
|  us-east-1f      |  192.168.0.0/19   |  subnet-01846036067209302  |
|  us-east-1f      |  192.168.64.0/19  |  subnet-0bc87ec1b4814bd4c  |
+------------------+-------------------+----------------------------+
-------------------------------------------------------------------------------------

Need to edit only the subnet id obtained in the above steps and paste in the terminal because security group id and file system id has been stored as a variable in the above steps. Example below

aws efs create-mount-target \
    --file-system-id $file_system_id \
    --subnet-id subnet-0f589bc69b5cd355b \
    --security-groups $security_group_id

aws efs create-mount-target \
    --file-system-id $file_system_id \
    --subnet-id subnet-0bc87ec1b4814bd4c \
    --security-groups $security_group_id

------------------------------------------------------------------------------
Go to this link https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/docs/README.md#examples
we can find the example applications to deploy 

lets go with Dynamic provisioning Example

https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/examples/kubernetes/dynamic_provisioning/README.md


Step 1:- aws efs describe-file-systems --query "FileSystems[*].FileSystemId" --output text

Step 2:- curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/dynamic_provisioning/specs/storageclass.yaml

Step 3:- open the storageclass.yaml file and edit with the filesystem id
and k apply -f storageclass.yaml

k create deployment-pvc.yaml

k get pods && k get pvc

kubectl get pods -n kube-system | grep efs-csi-controller

kubectl logs efs-csi-controller-74ccf9f566-q5989 \
    -n kube-system \
    -c csi-provisioner \
    --tail 10

    (Change the pod name in the above logs command)

kubectl get pv && kubectl get pvc && kubectl get pods -o wide




    



    



