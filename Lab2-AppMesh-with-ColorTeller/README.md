This lab builds upon lab 1


## 5. Build the Docker Images and push them to ECR

Populate some environment variables that we will be using

```
STACK_NAME=eksctl-$CLUSTER-cluster

VPC_ID=$(aws cloudformation describe-stacks --stack-name "$STACK_NAME" | jq -r '[.Stacks[0].Outputs[] | {key: .OutputKey, value: .OutputValue}] | from_entries' | jq -r '.VPC')

AWS_ACCOUNT_ID=$(aws sts get-caller-identity | jq -r '.Account')

```

First create the ECR repositories for the 2 applications.

```
aws ecr create-repository --repository-name colorteller

aws ecr create-repository --repository-name colorgateway
```

In the terminal of Cloud9, clone the code

```
git clone https://github.com/tohwsw/aws-app-mesh-examples.git
```

Retrieve the login command to use to authenticate your Docker client to your registry.

```
$(aws ecr get-login --no-include-email --region ap-southeast-1)
```

Go to the folder examples/apps/colorapp/src/colorteller. Execute a docker build with the respective repository uri for colorteller and push it to the repository.

```
cd ~/environment/aws-app-mesh-examples/examples/apps/colorapp/src/colorteller

docker build -t $AWS_ACCOUNT_ID.dkr.ecr.ap-southeast-1.amazonaws.com/colorteller .

docker push $AWS_ACCOUNT_ID.dkr.ecr.ap-southeast-1.amazonaws.com/colorteller:latest
```

Go to the folder examples/apps/colorapp/src/gateway. Execute a docker build with the respective repository uri for colorgateway and push it to the repository.

```
cd ~/environment/aws-app-mesh-examples/examples/apps/colorapp/src/gateway

docker build -t $AWS_ACCOUNT_ID.dkr.ecr.ap-southeast-1.amazonaws.com/colorgateway .

docker push $AWS_ACCOUNT_ID.dkr.ecr.ap-southeast-1.amazonaws.com/colorgateway:latest

```

## 6. Deploy the ALB Ingress Controller

We will deploy the ALB ingress controller for ingress-based load balancing to Fargate pods.

![img1]

[img1]:https://github.com/tohwsw/aws-eks-workshop-fargate/blob/master/Lab2-AppMesh-with-ColorTeller/img/ALB-Ingress-Controller-Fargate-architecture_pod.png

To get started, we’ll implement IAM roles for service accounts on our cluster in order to give fine-grained IAM permissions to our ingress controller pods.

First setup the OIDC ID provider (IdP) in AWS. This step is needed to give IAM permissions to a Fargate pod running in the cluster using the IAM for Service Accounts feature. Let’s setup the OIDC provider for your cluster it with the following command.

```
eksctl utils associate-iam-oidc-provider --cluster $CLUSTER --approve

```



The next step is to create the IAM policy that will be used by the ALB Ingress Controller deployment. This policy will be later associated to the Kubernetes service account and will allow the ALB Ingress Controller pods to create and manage the ALB’s resources in your AWS account for you.

```
wget -O alb-ingress-iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/master/docs/examples/iam-policy.json

aws iam create-policy --policy-name ALBIngressControllerIAMPolicy --policy-document file://alb-ingress-iam-policy.json

```

To create a service account, run the following command:

```
eksctl create iamserviceaccount --name alb-ingress-controller --namespace kube-system --cluster $CLUSTER --attach-policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/ALBIngressControllerIAMPolicy --approve --override-existing-serviceaccounts

STACK_NAME=eksctl-$CLUSTER-addon-iamserviceaccount-kube-system-alb-ingress-controller

ROLE_ARN=$(aws cloudformation describe-stacks --stack-name "$STACK_NAME" | jq -r '[.Stacks[0].Outputs[] | {key: .OutputKey, value: .OutputValue}] | from_entries' | jq -r '.Role1')

```

create RBAC permissions and a service account for the ALB Ingress Controller

```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/rbac-role.yaml

```

Print out the role that was created earlier

```
echo $ROLE_ARN

```

Open the rbac-role.yaml file in a text editor, and then make the following changes only to the ServiceAccount section.

```
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/name: alb-ingress-controller
  annotations:                                                                        # Add the annotations line
    eks.amazonaws.com/role-arn: arn:aws:iam::<AWS_ACCOUNT_ID>:role/<IAM_ROLE_NAME>    # Add the IAM role
  name: alb-ingress-controller
  namespace: kube-system

 ```

Save the rbac-role.yaml file, and then run the following command:

```
kubectl apply -f rbac-role.yaml

```

Download the file to set up the controller

```

curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.5/docs/examples/alb-ingress-controller.yaml

```

Make the following changes

```

    spec:
      containers:
      - args:
        - --ingress-class=alb
        - --cluster-name=$CLUSTER	               	#<-- Add the cluster name
        - --aws-vpc-id=$VPC_ID	         		#<-- Add the VPC ID 
        - --aws-region=ap-southeast-1			        	#<-- Add the region 
        image: docker.io/amazon/aws-alb-ingress-controller:v1.1.5	#<======= Please make sure the Image is 1.1.4 and above. 
        imagePullPolicy: IfNotPresent

```

To apply the alb-ingress-controller.yaml file, run the following command:

```
kubectl apply -f alb-ingress-controller.yaml

```

To check the status of the alb-ingress-controller deployment, run the following command:

```

kubectl rollout status deployment alb-ingress-controller -n kube-system


```

## 7. Deploy the application

To deploy the app, download colorapp.yml and and deploy it.

```
curl -O https://raw.githubusercontent.com/tohwsw/aws-eks-workshop-fargate/master/Lab2-AppMesh-with-ColorTeller/colorapp.yml

```

**Important** You will need to update color.yaml with the ECR respository uri of your colorgateway and colorteller docker images.
You can do so by substituting the account id 284245693010 with your own.

```
sed -i 's/284245693010/$AWS_ACCOUNT_ID/g' colorapp.yml

```

Next, deploy the applications.


```
kubectl apply -f colorapp.yml

```

Next, deploy the ingress

```
curl -O https://raw.githubusercontent.com/tohwsw/aws-eks-workshop-fargate/master/Lab2-AppMesh-with-ColorTeller/colorapp-ingress.yml

kubectl apply -f colorapp-ingress.yml

```

You can verify that the pods are running properly.

```
kubectl get pods

```


## 8. Test the application

An application load balancer is also deployed. Identify the address of the alb via the command.

```
export ALB=$(kubectl get ingress -o wide |grep colorapp-ingress | awk '{print $3}')

```


Next, paste the following to request the colorgateway service. Do note that the alb might take a few minutes to become active.

```
while [ 1 ]; do  curl -s --connect-timeout 2 $ALB:/color;echo;sleep 1; done

```

You should see the default color white is returned on each request.


## 9. Lab Cleanup

To clean up the lab, please delete using the following command

```
eksctl delete cluster --name=$CLUSTER --region=ap-southeast-1

```

## 10. Optional Labs

There are more labs hosted at https://eksworkshop.com/. Do check them out!








