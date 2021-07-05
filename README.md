## Instuctions
* Creating IAM Roles
```
#Kindly note down the ARNs of all the roles which we are going to create now
# Role for EKS Cluster
aws iam create-role --role-name EKS-Cluster-Role  --assume-role-policy-document file://eks_role.json
aws iam attach-role-policy --role-name EKS-Cluster-Role --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

# Role for EKS Nodes
aws iam create-role --role-name EKS-Node-Role  --assume-role-policy-document file://eks_node_role.json
aws iam attach-role-policy --role-name EKS-Node-Role --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
aws iam attach-role-policy --role-name EKS-Node-Role --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
aws iam attach-role-policy --role-name EKS-Node-Role --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

#Role to Access EKS Cluster from EC2 Instance
aws iam create-role --role-name EKS-Access-Role  --assume-role-policy-document file://eks_node_role.json
aws iam create-policy --policy-name EKS-Access-Policy --policy-document file://access_policy.json
#After running the above command, kindly note down the ARN of the policy created
aws iam attach-role-policy --role-name EKS-Access-Role --policy-arn PASTE_YOUR_COPIED_ARN
```
* Setting Up EKS Cluster
```
#Creating EKS Cluster
aws eks create-cluster --region <REGION_CDOE> --name <CLUSTER_NAME> --kubernetes-version <VERSION> --role-arn <ARN_EKS_CLUSTER_ROLE> --resources-vpc-config subnetIds=<YOUR_SUBNET_IDS_Separated_by_comma>
#Adding Node Group
aws eks create-nodegroup --cluster-name <CLUSTER_NAME> --nodegroup-name <NODE_GROUP_NAME> --scaling-config minSize=1,maxSize=1,desiredSize=1 --node-role <ARN_EKS_NODE_ROLE> --subnets <YOUR_SUBNET_IDS_Separated_by_space> --region us-west-2
```

* Accessing EKS Cluster
```
#Get KubeConfig
aws eks update-kubeconfig --region <YOUR_REGION> --name <YOUR_CLUSTER_NAME>
#Verify Access by running a Kubectl Command"
kubectl get nodes
```
* Attaching IAM Role to the EC2 instance
```
#Make sure your EC2 instance have the required IAM Role attached to it, if not you can attach it using below set of command
#1. Create Instance Profile
aws iam create-instance-profile --instance-profile-name EKS_Access_Profile
#2. Attach role to Instance Profile
aws iam add-role-to-instance-profile --role-name EKS-Access-Role --instance-profile-name EKS_Access_Role
aws ec2 associate-iam-instance-profile --region <YOUR_REGION> --instance-id <YOUR_INSTANCE_ID> --iam-instance-profile Name=EKS-Access-Role
```
* Configure EKS to get accessed from EC2 Machine using IAM Role
  * To edit aws-auth ConfigMap
  ```kubectl edit configmap aws-auth -n kube-system```
  * To add an IAM role in Config Map, put the following snippet in the file under mapRoles. Moreover, replace rolearn with your EKS-Access-Role ARN and username with role name
  ```
    - rolearn: arn:aws:iam::XXXXXXXXXXXX:role/testrole
      username: testrole
      groups:
        - system:masters
  ```
  * Note: The system:masters group allows superuser access to perform any action on any resource. F
* Accessing the EKS Cluster from EC2
  * To update or generate the kubeconfig file after aws-auth ConfigMap is updated
  ```aws eks update-kubeconfig --region <YOUR_REGION> --name <YOUR_CLUSTER_NAME>```
  * To verify the update of kubeconfig
  ```kubectl config view --minify```
  * That's it, you are now accesing the EKS Cluster using the IAM Role
  * Let's deploy a sample application on the cluster now
  ```kubectl create deployment thinknyx-app --image thechetantalwar/php --replicas 2```
  * You can verify the deployment by looking at the pod status
  ```kubectl get pods```
  * Expose the deployment using Kubernetes Service (LoadBalancer)
  ```kubectl expose deployment thinknyx-app --name thinknyx-service --port 80 --type LoadBalancer```
  * Let's check the service status
  ```kubectl get service```
  * Note the LoadBalancer address from the above stated command, and access the app in browser
* Happy Learning