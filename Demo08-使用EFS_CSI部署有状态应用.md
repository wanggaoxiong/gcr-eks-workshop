# Demo08-使用EFS CSI部署有状态应用
--
#### Contributor: Tao Dai
--

### 1 创建IAM策略和角色

1.1 配置环境变量

``` 
CLUSTER_NAME=prod
AWS_REGION=cn-northwest-1
```

1.2 创建EFS CSI所需的IAM策略

a.将IAM策略文件下载到本地

```
curl -o iam-policy-example.json https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/v1.3.2/docs/iam-policy-example.json
```
b.创建IAM策略

```
POLICY_ARN=$(aws iam create-policy \
    --policy-name AmazonEKS_EFS_CSI_Driver_Policy \
    --policy-document file://iam-policy-example.json \
    --query 'Policy.Arn' \
    --output text)
```
1.3 创建IAM Service Account

```
eksctl create iamserviceaccount \
    --name efs-csi-controller-sa \
    --namespace kube-system \
    --cluster $CLUSTER_NAME \
    --attach-policy-arn $POLICY_ARN \
    --approve \
    --override-existing-serviceaccounts \
    --region $AWS_REGION
```
### 2 安装EFS CSI驱动程序

2.1 下载驱动程序配置文件

```
kubectl kustomize \
    "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.3" > public-ecr-driver.yaml
```
2.2 修改驱动程序配置文件，去掉Service Account部分

```
sed -i '1,8d' public-ecr-driver.yaml
```
2.3 安装EFS CSI驱动程序

```
kubectl apply -f public-ecr-driver.yaml
```

### 3 创建Amazon EFS文件系统

3.1 创建EFS所需的网络资源


a.获取EKS集群所属VPC ID

```
vpc_id=$(aws eks describe-cluster \
    --name $CLUSTER_NAME \
    --query "cluster.resourcesVpcConfig.vpcId" \
    --output text)
```
b.获取VPC CIDR

```
cidr_range=$(aws ec2 describe-vpcs \
    --vpc-ids $vpc_id \
    --query "Vpcs[].CidrBlock" \
    --output text)
```
c.创建EFS文件系统使用的安全组

```
security_group_id=$(aws ec2 create-security-group \
    --group-name MyEfsSecurityGroup \
    --description "My EFS security group" \
    --vpc-id $vpc_id \
    --output text)
```

d.在EFS安全组中添加入站规则

```
aws ec2 authorize-security-group-ingress \
    --group-id $security_group_id \
    --protocol tcp \
    --port 2049 \
    --cidr $cidr_range
```
3.2 创建EFS文件系统

```
file_system_id=$(aws efs create-file-system \
    --region $AWS_REGION \
    --performance-mode generalPurpose \
    --query 'FileSystemId' \
    --output text)
```

3.3 为EFS文件系统添加挂载目标

a. 获取EKS所属VPC中的私有子网

```
aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$vpc_id" "Name=tag:kubernetes.io/role/internal-elb,Values=1"  \
    --query 'Subnets[].SubnetId' \
    --output json > subnets_list.txt
```

b.对子网清单文件进行处理

```
sed -i '1d;$d' subnets_list.txt
sed -i 's/\"//g' subnets_list.txt
sed -i 's/\,//g' subnets_list.txt
```

c.在EFS文件系统中创建挂载目标

```
while read -r line
do
  aws efs create-mount-target \
    --file-system-id $file_system_id \
    --subnet-id $line \
    --security-groups $security_group_id
done < subnets_list.txt
```
### 4 部署示例应用程序

4.1 创建StorageClass

a.获取EFS文件系统ID

```
file_system_id=$(aws efs describe-file-systems --query "FileSystems[*].FileSystemId" --output text)
```

b.下载StorageClass配置文件

```
curl -o storageclass.yaml https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/dynamic_provisioning/specs/storageclass.yaml
```

c.修改文件系统ID

```
sed -i '8c \  fileSystemId: '$file_system_id'' storageclass.yaml
```

d.部署StorageClass

```
kubectl apply -f storageclass.yaml
```
4.2 部署示例应用程序

a. 下载示例应用程序配置文件

```
curl -o pod.yaml https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/dynamic_provisioning/specs/pod.yaml
```
b. 部署示例应用程序

```
kubectl apply -f pod.yaml
```
4.3 验证示例应用程序

a. 查看Pod

```
kubectl get pods
efs-app   1/1     Running   0          17s
```
b.查看PV, PVC

```
kubectl get pv,pvc | grep efs
persistentvolume/pvc-c55e2c43-a8e7-494d-a0e6-f3c6e16f1e3c   5Gi        RWX            Delete           Bound    default/efs-claim   efs-sc                  69s
persistentvolumeclaim/efs-claim   Bound    pvc-c55e2c43-a8e7-494d-a0e6-f3c6e16f1e3c   5Gi        RWX            efs-sc         69s
```

c.确认数据已经写入EFS文件系统

```
kubectl exec efs-app -- bash -c "cat data/out"
Wed Jul 6 07:14:04 UTC 2022
Wed Jul 6 07:14:09 UTC 2022
Wed Jul 6 07:14:14 UTC 2022
Wed Jul 6 07:14:19 UTC 2022
Wed Jul 6 07:14:24 UTC 2022
Wed Jul 6 07:14:29 UTC 2022
Wed Jul 6 07:14:34 UTC 2022
```
### 5 清理环境

```
# 删除示例应用程序
kubectl delete -f pod.yaml

# 删除EFS CSI驱动程序
kubectl delete -f pod.yaml

# 删除EFS文件系统
aws efs delete-file-system \
    --file-system-id $file_system_id

# 删除IAM Service Account
eksctl delete iamserviceaccount \
  --name efs-csi-controller-sa \
  --cluster $CLUSTER_NAME

# 删除IAM Policy
aws iam delete-policy --policy-arn $POLICY_ARN
```













