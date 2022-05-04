# vpc-cni
Custom networking with the AWS VPC CNI plug-in

High Level Steps:
1. Attach secondary CIDR to existing VPC where EKS cluster is created
2. Create Subnets within secondary CIDR range
3. Create eniconfig for each AZ and apply it
4. Add env variables (AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG,ENI_CONFIG_LABEL_DEF, AWS_VPC_K8S_CNI_EXTERNALSNAT) for aws-node daemonset 
5. Create eniconfig for each AZ and apply it
6. Configure Custom networking
7. TEST NETWORKING

Will have following network for VPC-CNI example:

          VPC          CIDR	             Subnet A	    Subnet B	    Subnet C
          Primary	192.168.0.0/16	192.168.0.0/19	192.168.32.0/19	192.168.64.0/19
          Secondary	100.64.0.0/16	100.64.0.0/19	100.64.32.0/19	100.64.64.0/19


**1.Attach secondary CIDR to existing VPC where EKS cluster is created**


          VPC_ID=aws ec2 describe-vpcs --filters Name=tag:Name,Values=*eksdemo-qa* --query 'Vpcs[].VpcId' --output text
          aws ec2 associate-vpc-cidr-block --vpc-id $VPC_ID --cidr-block 100.64.0.0/16

<img width="1265" alt="image" src="https://user-images.githubusercontent.com/74225291/166150682-ab6b202e-bd67-495e-894f-1ae447cb71ea.png">


**2.Create Subnets within secondary CIDR range**


I have 3 instances and using 3 subnets in my environment. For simplicity, we will use the same AZ’s and create 3 secondary CIDR subnets but you can certainly customize according to your networking requirements.


          export POD_AZS=($(aws ec2 describe-instances --filters "Name=tag-key,Values=eks:cluster-name" "Name=tag-value,Values=*eksdemo-qa*" --query 'Reservations[*].Instances[*].[Placement.AvailabilityZone]' --output text | sort | uniq))
          
          
          CGNAT_SNET1=$(aws ec2 create-subnet --cidr-block 100.64.0.0/19 --vpc-id $VPC_ID --availability-zone us-east-2a --query 'Subnet.SubnetId' --output text)
          CGNAT_SNET2=$(aws ec2 create-subnet --cidr-block 100.64.32.0/19 --vpc-id $VPC_ID --availability-zone us-east-2b --query 'Subnet.SubnetId' --output text)
          CGNAT_SNET3=$(aws ec2 create-subnet --cidr-block 100.64.64.0/19 --vpc-id $VPC_ID --availability-zone us-east-2c --query 'Subnet.SubnetId' --output text)

we need to associate three new subnets into a route table. Again for simplicity, we chose to add new subnets to the Public route table that has connectivity to Internet Gateway

          SNET1=$(aws ec2 describe-subnets --filters Name=cidr-block,Values=192.168.0.0/19 --query 'Subnets[].SubnetId' --output text)
          RTASSOC_ID=$(aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=$SNET1 --query 'RouteTables[].RouteTableId' --output text)
          
          aws ec2 associate-route-table --route-table-id $RTASSOC_ID --subnet-id $CGNAT_SNET1
          aws ec2 associate-route-table --route-table-id $RTASSOC_ID --subnet-id $CGNAT_SNET2
          aws ec2 associate-route-table --route-table-id $RTASSOC_ID --subnet-id $CGNAT_SNET3



**3.CONFIGURE CNI**

Before we start making changes to VPC CNI, let’s make sure we are using latest CNI version

Run this command to find CNI version

          kubectl describe daemonset aws-node --namespace kube-system | grep Image | cut -d "/" -f 2
          amazon-k8s-cni-init:v1.10.1-eksbuild.1
          amazon-k8s-cni:v1.10.1-eksbuild.1

CNI version must be > 1.7.5

**4.CREATE CRDS**

You should have ENIConfig CRD already installed with latest CNI version (1.3+). You can check if its installed by running this command.

          # kubectl get crd
          NAME                                         CREATED AT
          eniconfigs.crd.k8s.amazonaws.com             2022-05-01T12:18:09Z

If you don’t have ENIConfig installed, you can install it by using this command

          kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/release-1.7/config/v1.7/aws-k8s-cni.yaml

**5.Create eniconfig for each AZ and apply it**

Check your Worker Node SecurityGroup

          # INSTANCE_IDS=(`aws ec2 describe-instances --query 'Reservations[*].Instances[*].InstanceId' --filters "Name=tag-key,Values=eks:cluster-name" "Name=tag-value,Values=*eksdemo*" --output text`)
          # export NETCONFIG_SECURITY_GROUPS=$(for i in "${INSTANCE_IDS[@]}"; do  aws ec2 describe-instances --instance-ids $i | jq -r '.Reservations[].Instances[].SecurityGroups[].GroupId'; done  | sort | uniq | awk -vORS=, '{print $1 }' | sed 's/,$//')
          # echo $NETCONFIG_SECURITY_GROUPS
          sg-0917c42587e776c96,sg-0dbb27eab2c02382a
          
          # aws ec2 describe-subnets  --filters "Name=cidr-block,Values=100.64.*" --query 'Subnets[*].[CidrBlock,SubnetId,AvailabilityZone]' --output table
          --------------------------------------------------------------
          |                       DescribeSubnets                      |
          +-----------------+----------------------------+-------------+
          |  100.64.0.0/19  |  subnet-0ba2c12d4c21e763a  |  us-east-2a |
          |  100.64.32.0/19 |  subnet-0899f7eb01fd459e7  |  us-east-2b |
          |  100.64.64.0/19 |  subnet-06c992061fd4e98f2  |  us-east-2c |
          +-----------------+----------------------------+-------------+


Create eniconfig as per AZ and apply it.

          # kubectl apply -f us-east-2a.yaml 
          eniconfig.crd.k8s.amazonaws.com/us-east-2a created
          # kubectl apply -f us-east-2b.yaml 
          eniconfig.crd.k8s.amazonaws.com/us-east-2b created
          # kubectl apply -f us-east-2c.yaml 
          eniconfig.crd.k8s.amazonaws.com/us-east-2c created

**6.Configure Custom networking**

          kubectl set env daemonset aws-node -n kube-system AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true ENI_CONFIG_LABEL_DEF=failure-domain.beta.kubernetes.io/zone AWS_VPC_K8S_CNI_EXTERNALSNAT=false
          daemonset.apps/aws-node env updated


After this you will see that aws-node pods are getting restarted.

          # kubectl get pods -n kube-system|grep -i aws-node
          NAMESPACE     NAME                       READY   STATUS              RESTARTS   AGE
          kube-system   aws-node-ctdjt             1/1     Running             0          11s
          kube-system   aws-node-hxrpg             1/1     Running             0          24s
          kube-system   aws-node-j2zkk             1/1     Terminating         0          7m43s

**7.TEST NETWORKING**

Launch pods into Secondary CIDR network

          kubectl create deployment nginx --image=nginx
          kubectl scale --replicas=3 deployments/nginx
          kubectl expose deployment/nginx --type=NodePort --port 80
          kubectl get pods -o wide

          # kubectl get pods -o wide -A
          NAMESPACE     NAME                       READY   STATUS    RESTARTS   AGE     IP             
          default       nginx-6799fc88d8-6w7j7     1/1     Running   0          8m44s   100.64.91.36   
          default       nginx-6799fc88d8-9j9s8     1/1     Running   0          8m45s   100.64.72.146  
          default       nginx-6799fc88d8-gmhv9     1/1     Running   0          8m45s   100.64.43.160  
          kube-system   aws-node-5cwzh             1/1     Running   0          7m22s   192.168.5.237  
          kube-system   aws-node-9clnc             1/1     Running   0          5m36s   192.168.73.215 
          kube-system   aws-node-psbfp             1/1     Running   0          3m40s   192.168.41.255 
          kube-system   coredns-79ddc48959-xkx7v   1/1     Running   0          8m44s   100.64.12.25   
          kube-system   coredns-79ddc48959-xrn66   1/1     Running   0          8m45s   100.64.1.100   
          kube-system   kube-proxy-cft2t           1/1     Running   0          7m22s   192.168.5.237  
          kube-system   kube-proxy-hxchf           1/1     Running   0          5m36s   192.168.73.215 
          kube-system   kube-proxy-mglfb           1/1     Running   0          3m40s   192.168.41.255 

Additional notes on ‘maxPodsPerNode’ :

Enabling a custom network effectively removes an available network interface (and all of its available IP addresses for pods) from each node that uses it. The primary network interface for the node is not used for pod placement when a custom network is enabled. Determine the maximum number of pods that can be scheduled on each node using the following formula.


           maxPodsPerNode = (number of interfaces - 1) * (max IPv4 addresses per interface - 1) + 2

           For example, use the following value for t3.small
           maxPodsPerNode = (3 - 1) * (4 - 1) + 2 = 8
