The file shows how to create the first Kubernetes cluster on AWS EC2 instances step by step. 
Refer to [kops](https://github.com/alvenwong/kops/blob/master/docs/aws.md).
# Dependencies
## Install kops
Refer to [kops](https://github.com/kubernetes/kops/blob/master/docs/install.md).
```bash
wget -O kops https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
chmod +x ./kops
sudo mv ./kops /usr/local/bin/
```
## Install Kubectl
Refer to [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/). <p>
Since we will install minikubte for monitoring while Amazon Linux CentOS does not support systemctl, 
the kernel version for this installation is Ubuntu.
```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
```
## Install AWS CLI
```bash
pip install awscli
```

# Create an IAM user
Create an IAM user for kops

```bash
aws iam create-group --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops
aws iam create-user --user-name kops
aws iam add-user-to-group --user-name kops --group-name kops
aws iam create-access-key --user-name kops
```
Record the SecretAccessKey and AccessKeyID in the returned JSON output, and then use them below:
```bash
# configure the aws client to use your new IAM user
aws configure           # Use your new access and secret key here
aws iam list-users      # you should see a list of all your IAM users here
# Because "aws configure" doesn't export these vars for kops to use, we export them now
export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)
export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)
```

# Configure DNS
## Create a subdomain
Buy a domain from [AWS](https://console.aws.amazon.com/route53/home). My domain is wangzhuabg93.com. 
Then create a subdomain and note the subdomain name servers. My subdomain is brown.
```bash
# Note: This example assumes you have jq installed locally.
ID=$(uuidgen) && aws route53 create-hosted-zone --name subdomain.wangzhuang93.com --caller-reference $ID | \
    jq .DelegationSet.NameServers
```
Note your PARENT hosted zone id
```bash
aws route53 list-hosted-zones | jq '.HostedZones[] | select(.Name=="wangzhuang93.com.") | .Id’
```
Create a new JSON file (subdomain.json) with your values.
```bash
{
  "Comment": "Create a subdomain NS record in the parent domain",
  "Changes": [
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "brown.wangzhuang93.com",
        "Type": "NS",
        "TTL": 300,
        "ResourceRecords": [
          {
            "Value": "ns-1.awsdns-1.co.uk"
          },
          {
            "Value": "ns-2.awsdns-2.org"
          },
          {
            "Value": "ns-3.awsdns-3.com"
          },
          {
            "Value": "ns-4.awsdns-4.net"
          }
        ]
      }
    }
  ]
}
```
Replace "brown.wangzhuang93.com" with your subdomain and the four values with your subdomain name servers.<p>
Apply the SUBDOMAIN NS records to the PARENT hosted zone.
```bash
aws route53 change-resource-record-sets --hosted-zone-id parent-zone-id --change-batch file://subdomain.json
```
Replace parent-zone-id with your PARENT hosted zone id. You can check the name servers in 
[hosted zones](https://console.aws.amazon.com/route53/home#hosted-zones:)

## Test DNS setup
```bash
dig ns brown.wangzhuang93.com
```
Should return something similar to:
```bash
;; ANSWER SECTION:
subdomain.example.com.        172800  IN  NS  ns-1.awsdns-1.net.
subdomain.example.com.        172800  IN  NS  ns-2.awsdns-2.org.
subdomain.example.com.        172800  IN  NS  ns-3.awsdns-3.com.
subdomain.example.com.        172800  IN  NS  ns-4.awsdns-4.co.uk.
```

# Set cluster state storage 
```bash
aws s3api create-bucket --bucket brown-wangzhuang93-com-state-store --region us-east-1
aws s3api put-bucket-versioning --bucket brown-wangzhuang93-com-state-store  --versioning-configuration Status=Enabled
aws s3api put-bucket-encryption --bucket brown-wangzhuang93-com-state-store --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
```
Recommand to set the region as us-east-1. 

# Create the cluster
Set the names of the cluster and state store
```bash
export NAME=brown.wangzhuang93.com
export KOPS_STATE_STORE=s3://brown-wangzhuang93-com-state-store
```
Create the cluster
```bash
kops create cluster --zones us-east-2a ${NAME}
```
Build the cluster
```bash
kops update cluster ${NAME} --yes
```
Delete the cluster
```bash
kops delete cluster --name ${NAME} --yes
```

# Basic kubectl commands
```bash
kops validate cluster
kubectl get namespace
kubectl get services
kubectl get nodes -o wide
kubectl -n kube-system get po
kubectl --namespace="sock-shop" get pods 
kubectl create namespace sock-shop
kubectl apply -f complete-demo.yaml
kubectl delete namespace sock-shop
kubectl --namespace=sock-shop describe service
kubectl get svc --namespace=sock-shop -o wide
ssh -i ~/.ssh/id_rsa admin@api.brown.wangzhuang93.com
```

# Install benchmark and monitoring
## Install minikubte
Refer to [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/). 
Recommand to install minikubte on AWS EC2 Ubuntu because AWS EC2 CentOS doesn't support systemctl.
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  && sudo install minikube-linux-amd64 /usr/local/bin/minikube
```
## Install Zipkin
## Install benchmark
The microservices benchmark we choose to test is [Sock Shop](https://microservices-demo.github.io/).<p>
Clone the microservices-demo repository
```bash
git clone https://github.com/microservices-demo/microservices-demo
cd microservices-demo
```
Create a new namespace for this microservices demo.
```bash
kubectl create namespace sock-shop
```
Run this microservices demo
```bash
cd deploy/kubernetes/
kubectl apply -f complete-demo.yaml
```
Delete the demo
```bash
kubectl delete -f complete-demo.yaml
```
Delete the namespace
```bash
kubectl delete namespace sock-shop
```
## Load test
ssh to the master node of the cluster
```bash
ssh -i ~/.ssh/id_rsa admin@api.brown.wangzhuang93.com
```
Run the load test container
```bash
docker run --net=host weaveworksdemos/load-test -h $frontend-ip[:$port] -r 100 -c 2
```
You can get the ip and port for the frontend by running
```bash
kubectl get svc —namespace=sock-shop -o wide
```
