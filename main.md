## Create k8s cluster through kOps on AWS ( Amaznon linux 2)
### 1. pre-request 
#### 1.1) Inastall and configure awscli v2
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

complete -C '/usr/local/bin/aws_completer' aws; echo "complete -C '/usr/local/bin/aws_completer' aws" >> ~/.bashrc
source ~/.bash_profile
```
Check awscli version, 
```
aws --version
```
If the version of awscli is older(v1), you can follow the official link below to update awscli v2:  
[https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html#cliv2-linux-upgrade](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html#cliv2-linux-upgrade)  

configure credential file:
```
[ec2-user@ip-172-31-1-111 ~]$ aws configure
AWS Access Key ID [None]:  输入你的AK
AWS Secret Access Key [None]:  输入你的SK
Default region name [cn-northwest-1]: 此处默认为宁夏区域
Default output format [json]:  
```
Official link for configure aws cli:  
[https://docs.amazonaws.cn/en_us/cli/latest/userguide/cli-configure-files.html](https://docs.amazonaws.cn/en_us/cli/latest/userguide/cli-configure-files.html)


#### 1.2) install kubectl   
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/
```

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

### 2. install kops 
```
curl -Lo kops https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
chmod +x kops
sudo mv kops /usr/local/bin/kops
```

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

### 3. creat an configuire AWS IAM user 

#### 3.1) prepare AWS managed IAM policy  
```
[ec2-user@ip-10-250-31-74 ~]$ aws iam list-policies --query 'Policies[?PolicyName==`AmazonEC2FullAccess`].{ARN:Arn}' --output text. 
arn:aws-cn:iam::aws:policy/AmazonEC2FullAccess

[ec2-user@ip-10-250-31-74 ~]$ aws iam list-policies --query 'Policies[?PolicyName==`AmazonRoute53FullAccess`].{ARN:Arn}' --output text
arn:aws-cn:iam::aws:policy/AmazonRoute53FullAccess

[ec2-user@ip-10-250-31-74 ~]$ aws iam list-policies --query 'Policies[?PolicyName==`AmazonS3FullAccess`].{ARN:Arn}' --output text
arn:aws-cn:iam::aws:policy/AmazonS3FullAccess

[ec2-user@ip-10-250-31-74 ~]$ aws iam list-policies --query 'Policies[?PolicyName==`IAMFullAccess`].{ARN:Arn}' --output text
arn:aws-cn:iam::aws:policy/IAMFullAccess

[ec2-user@ip-10-250-31-74 ~]$ aws iam list-policies --query 'Policies[?PolicyName==`AmazonVPCFullAccess`].{ARN:Arn}' --output text
arn:aws-cn:iam::aws:policy/AmazonVPCFullAccess
```
ARN we need 
```
arn:aws-cn:iam::aws:policy/AmazonEC2FullAccess
arn:aws-cn:iam::aws:policy/AmazonRoute53FullAccess
arn:aws-cn:iam::aws:policy/AmazonS3FullAccess
arn:aws-cn:iam::aws:policy/IAMFullAccess
arn:aws-cn:iam::aws:policy/AmazonVPCFullAccess
```

#### 3.2) create IAM user  
```
aws iam create-user --user-name kops

aws iam attach-user-policy --policy-arn arn:aws-cn:iam::aws:policy/AmazonEC2FullAccess --user-name kops
aws iam attach-user-policy --policy-arn arn:aws-cn:iam::aws:policy/AmazonRoute53FullAccess --user-name kops
aws iam attach-user-policy --policy-arn arn:aws-cn:iam::aws:policy/AmazonS3FullAccess --user-name kops
aws iam attach-user-policy --policy-arn arn:aws-cn:iam::aws:policy/IAMFullAccess --user-name kops
aws iam attach-user-policy --policy-arn arn:aws-cn:iam::aws:policy/AmazonVPCFullAccess --user-name kops

aws iam list-attached-user-policies --user-name kops

#You should record the SecretAccessKey and AccessKeyID in the returned JSON output
aws iam create-access-key --user-name kops
aws configure  
aws sts get-caller-identity

export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)
export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)
```

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

### 4. Configure DNS 

#### 4.1) gossip-based DNS doesn't need this

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

### 5. Cluster state storage
```
aws s3api create-bucket --bucket jerry-example-com-state-store --region cn-northwest-1 --create-bucket-configuration LocationConstraint=cn-northwest-1
``` 
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

### 6. Creating your first cluster 

#### 6.1) prepare environment variables and generate cluster configuration
```
export NAME=myfirstcluster.k8s.local
export KOPS_STATE_STORE=s3://jerry-example-com-state-store

[ec2-user@ip-10-250-31-74 .aws]$ aws ec2 describe-availability-zones --region cn-northwest-1
{
    "AvailabilityZones": [
        {
            "State": "available",
            "OptInStatus": "opt-in-not-required",
            "Messages": [],
            "RegionName": "cn-northwest-1",
            "ZoneName": "cn-northwest-1a",
            "ZoneId": "cnnw1-az1",
            "GroupName": "cn-northwest-1",
            "NetworkBorderGroup": "cn-northwest-1",
            "ZoneType": "availability-zone"
        },
        {
            "State": "available",
            "OptInStatus": "opt-in-not-required",
            "Messages": [],
            "RegionName": "cn-northwest-1",
            "ZoneName": "cn-northwest-1b",
            "ZoneId": "cnnw1-az2",
            "GroupName": "cn-northwest-1",
            "NetworkBorderGroup": "cn-northwest-1",
            "ZoneType": "availability-zone"
        },
        {
            "State": "available",
            "OptInStatus": "opt-in-not-required",
            "Messages": [],
            "RegionName": "cn-northwest-1",
            "ZoneName": "cn-northwest-1c",
            "ZoneId": "cnnw1-az3",
            "GroupName": "cn-northwest-1",
            "NetworkBorderGroup": "cn-northwest-1",
            "ZoneType": "availability-zone"
        }
    ]
}

kops create cluster --zones=cn-northwest-1c ${NAME}

[ec2-user@ip-10-250-31-74 ~]$  aws s3 ls  s3://jerry-example-com-state-store/myfirstcluster.k8s.local/
                           PRE clusteraddons/
                           PRE instancegroup/
2021-08-06 13:52:49       5879 cluster.spec
2021-08-06 13:52:49       1206 config

kops edit cluster ${NAME}

kops update cluster ${NAME} --yes
```

#### 6.2) update kube config in ~/.kube/config and kubectl kube info
```
kops export kubecfg --admin

[ec2-user@ip-10-220-13-6 ~]$ kubectl get nodes
NAME                            STATUS   ROLES                  AGE   VERSION
ip-172-20-43-146.ec2.internal   Ready    control-plane,master   50m   v1.21.3
ip-172-20-44-47.ec2.internal    Ready    node                   49m   v1.21.3
[ec2-user@ip-10-220-13-6 ~]$ kubectl get all --all-namespaces
NAMESPACE     NAME                                                        READY   STATUS    RESTARTS   AGE
kube-system   pod/coredns-autoscaler-6f594f4c58-5ghgc                     1/1     Running   0          50m
kube-system   pod/coredns-f45c4bf76-9t5tm                                 1/1     Running   0          48m
kube-system   pod/coredns-f45c4bf76-swrxp                                 1/1     Running   0          50m
kube-system   pod/dns-controller-5d59c585d8-jhxz2                         1/1     Running   0          50m
kube-system   pod/etcd-manager-events-ip-172-20-43-146.ec2.internal       1/1     Running   0          49m
kube-system   pod/etcd-manager-main-ip-172-20-43-146.ec2.internal         1/1     Running   0          49m
kube-system   pod/kops-controller-gq5v6                                   1/1     Running   0          49m
kube-system   pod/kube-apiserver-ip-172-20-43-146.ec2.internal            2/2     Running   0          49m
kube-system   pod/kube-controller-manager-ip-172-20-43-146.ec2.internal   1/1     Running   0          49m
kube-system   pod/kube-proxy-ip-172-20-43-146.ec2.internal                1/1     Running   0          49m
kube-system   pod/kube-proxy-ip-172-20-44-47.ec2.internal                 1/1     Running   0          48m
kube-system   pod/kube-scheduler-ip-172-20-43-146.ec2.internal            1/1     Running   0          49m

NAMESPACE     NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes   ClusterIP   100.64.0.1    <none>        443/TCP                  50m
kube-system   service/kube-dns     ClusterIP   100.64.0.10   <none>        53/UDP,53/TCP,9153/TCP   50m

NAMESPACE     NAME                             DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                                                      AGE
kube-system   daemonset.apps/kops-controller   1         1         1       1            1           kops.k8s.io/kops-controller-pki=,node-role.kubernetes.io/master=   50m

NAMESPACE     NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/coredns              2/2     2            2           50m
kube-system   deployment.apps/coredns-autoscaler   1/1     1            1           50m
kube-system   deployment.apps/dns-controller       1/1     1            1           50m

NAMESPACE     NAME                                            DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/coredns-autoscaler-6f594f4c58   1         1         1       50m
kube-system   replicaset.apps/coredns-f45c4bf76               2         2         2       50m
kube-system   replicaset.apps/dns-controller-5d59c585d8       1         1         1       50m
```


#### reference: 
[https://kops.sigs.k8s.io/getting_started/install/](https://kops.sigs.k8s.io/getting_started/install/)
