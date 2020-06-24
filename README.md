# k8s-deployment

## About

This document briefly describes how to deploy and scale Nodejs application on Kubernetes running on the AWS cloud. It also includes, how we can autoscale our applications in Kubernetes to achieve high availability. 

## Prerequisite

1) Create two EC2 Linux instances on AWS with a minimum configuration of 2 vCPU and 4GB of RAM and name them as Master and Node. 
2) Create one AWS IAM user with necessary permission, note down ACCESS_KEY, and SECRET_KEY of IAM USER. This credentials we will use to log in on AWS.
      
3) Create one ECR repository with the name Nodejs-test on AWS.  
    
4) Setup a Kubernetes cluster on AWS using the kubeadmin tool. Please refer the following link for reference:
   https://www.edureka.co/blog/install-kubernetes-on-ubuntu
   
   https://vitux.com/install-and-deploy-kubernetes-on-ubuntu/

   Note: You can also refer to the Kubernetes official documentation. 

## Description 

1) Clone the above application from Github on AWS Master EC2 instance using the following command:
```
git clone https://github.com/Aaditya7789/k8s-deployment.git
```
    
2) Then switch to `k8s-deployment` directory. Here you will see all files and folders related to your deployment.
    
3) For deploying an application on the k8s you need a docker image stored in any private or public docker registries like  Dockerhub, AWS ECR (Elastic container registry), or GCR (Google container registry). 
   
4) As mentioned above you need a Docker image to deploy your application. This we can get by building an image from Dockerfile stored in `nodejs-app` directory. 
   
5) Before building the docker image we need to login to AWS from our terminal using aws-cli. Download the aws-cli on your terminal using the following link:
   
   https://aws.amazon.com/cli/
    
6) For login use, the same AWS ACCESS_KEY & SECRET_KEY generated at the time of user creation. 
   
7) Once you logged in to build the docker image and push it to AWS ECR private registry using following commands:

Retrieve an authentication token and authenticate your Docker client to your registry.
Use the AWS CLI:
```
aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin ACCOUNT_ID.dkr.ecr.us-east-2.amazonaws.com
```
Build your Docker image using the following command. You can skip this step if your image is already built:
```
docker build -t nodejs-test .
```
After the build completes, tag your image so you can push the image to this repository:
```
docker tag nodejs-test:latest ACCOUNT_ID.dkr.ecr.us-east-2.amazonaws.com/nodejs-test:latest
```
Run the following command to push this image to your newly created AWS repository:
```
docker push ACCOUNT_ID.dkr.ecr.us-east-2.amazonaws.com/nodejs-test:latest
```
8) The above steps help you to build and push your images on AWS ECR. Each time when you make any new change in the code, first you need to build your image and then push that image to private registry like AWS ECR.


## Deployment 


This is the most crucial part of this project, here you need to understand and configure so many things to make your deployment successful. 

The first task is how to create pods and deploy applications on pods. 

Here, I have written a nodejs-deployment.yaml which contains lots of fo things in it. Let's understand in depth. 
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-app-deployment
  labels:
    app: nodejs-app
```

The above section is the general section to create any Kubernetes object. Which contains the version of Kubernetes API for specific object `apps/v1`, object type **Deployment**, and name of the object along with its labels.
 
 labels are generally used to identify the particular object and refer that object in another object creation. 
```yaml
 spec:
  replicas: 7
  selector:
    matchLabels:
      app: nodejs-app
  template:
    metadata:
      name: nodejs-pod
      labels:
        app: nodejs-app
    spec:
      containers:
      - name: nodejs-app
        image: $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/nodejs-test:latest
        ports:
          -  containerPort: 3000
        resources:
          limits:
            cpu: 300m
          requests:
            cpu: 150m
          limits:
            memory: "400Mi"
          requests:
            memory: "40Mi"
        imagePullPolicy: Always
      imagePullSecrets:
      -  name: ecr-registry-keycc:paytm-assignment
```
The spec section is the main section of the Deployment object. Which contains more information than above. 

1) `replicas`: This is to ensure the desired no. of pods available in the cluster after deployment. 
   
2) `selector`: It is used to select appropriate pod for deployment by referring the labels. 
   
3) `template`: This helps to create pods and containers inside it by referring to various parameters and requirements. 
   
This contains the name of the pods, name of the container, name of the image, name of the secret to pulling images from a private registry, the port where the application listens, and resources section to allocate memory and CPU from the host machine.   

I am assuming that you are aware of some of these like pods, containers, ports, and images. Let me explain to you others which is not commonly used i.e resources and secrets. 

`resources`: As I mentioned above this section used to allocate system resources to Kubernetes pods. Here I am allocating CPU and Memory. The reason behind to allocate these resources to pods is to autoscale the application pods based on CPU and Memory utilization. I have mentioned in the above limits=300m and requests=150m. This means I am allocating 0.3 Millicore CPU from my worker node. So pods can't consume more than this limit. And I am requesting 0.15 Millicore CPU to run my application from this allocated limit.  

The same thing applies to memory also. But here unit will change as `Mebibyte` and allocating Limits for the memory from system and request the memory used by all running pods in the deployment.

`imagePullSecrets`: This term contains the secret which can be used to pull image from private docker registry like AWS ECR in our case. Secret are generally containing the information of your credentials. We used a secret to store our private registry name, username, and password for login. 

If you use the default private docker registry (docker hub) then it will be easy to manage secrets. But this is not the case with the AWS ECR registry. Because AWS ECR will change its credentials in every 12 hours. Means your secret will expire after 12 hours. And you can't be used that secret anymore. To resolve this issue, I have created one shell script which can be used to rotate your secret before it expires. Please refer to the following details:

`secret-rotation-cron.sh`: This file contains a small shell script, which helps to pull the latest credentials from the ECR registry, delete the old secret, and create the new one. 

Let me explain few lines,
```
aws ecr get-login >>  /home/ubuntu/ecr-login.sh
```
In this command, we are pulling the latest credentials from the AWS ECR registry and storing it in another `ecr-login.sh` file. This `ecr-login.sh`  used to store and rotate these credentials.
```
kubectl delete secret ecr-registry-key >> cron-output.txt
```
In this command, we are deleting the old secret and storing the output of the command in `cron-output.txt`. Using this we can make sure our old secret has been successfully deleted. 
```
kubectl create secret generic ecr-registry-key --from-file=.dockerconfigjson=.docker/config.json --type=kubernetes.io/dockerconfigjson >> cron-output.txt
```
The above command is the main command to create a secret by referring `.docker/config.json`. Because each time when you pull the credentials by default its stores in `.docker/config.json` file which is the default config file of docker. 


Set the cron job for your `secret-rotaion-cron.sh` file using below cron 
```
0 */10 * * *      /home/ubuntu/k8s-deployment/secret-rotaion-cron.sh
```
This cron helps to rotate secret at every 10 hour. 


After, Performing these many steps we are ready to deploy our application.  

```
kuebectl create -f nodejs-deployment.yaml
```
This command will create the deployment on our Kubernetes cluster. 

Need to expose application on the worker node. for this we have created `nodejs-service.yaml` file which can help to expose application on node port. 

In `nodejs-service.yaml` the file we have mention port of the container where the application listing on, Service port which is similar to container port and helps to receive incoming traffic from NodePort and forward towards the container and NodePort which receives the incoming traffic. 

As per the Kubernetes documentation, Kubernetes nodes can serve the applications between port ranges 30000 to 32767. Hence, we have used `30080` port for exposing our application on the worker node.  

We have also configured an AWS classic load balancer for exposing the application on port 3000 instead of serving from Kubernetes default port range. We have open 3000 ports on the classic load balancer security group. which can be accessed by any location to browse the application. 

## Horizontal Pod Autoscaling  

In this section we discuss about how to autoscale our pods on the basis of different metircs. Kubernetes autoscales the pods by referring the metrics of pod and this metrics can be found with the help of metrics-server. 
That means we need one metrics server which can monitore the health of our kubernetes resources and help to autoscale our applications. 

To acheive the autoscaling in our current deployment we have deployed default kubernetes metrics server with the help of below metrics server yaml file. 

`metrics-server.yaml`: This file containes all neccessary details to deploy metric server. This file including the various kubernetes objects which can be used by metric server obeject to run and monitore in a flexible way. This includes object like clusterRole, clusterRoleBinding, service and API versions. 

To deploy metric server used the below command:
```
kubectl create -f metrics-server.yaml
```
By default it will deploy under kubernetes default namespace which is kube-system. You can also find metric server pod by executing below commands:
```
kubectl -n kube-system get pods
```
It will list the no. of pods present under default namespace. 

Metric server also create some other kubernetes objects which you can find by executing below command. 
```
kubectl get all --all-namespaces
```
This will list all kubernetes objects under all namespaces. 

To find the metrics of the cluster use below commands:

`kubectl top nodes`: It will list the nodes CPU and Memory utiliztion. 


`kubectl top pods`: It will list the Pods CPU and Memory utiliztion.

After deploying and validating the metrics server we will implement autoscaling for in our kubernetes cluster. 

To implemenet horizontal pod autoscaling we need to created horizontal pod autoscaler object which can be associate with our deployment. For this we have created two diffrent yaml files to perform autoscaling based on CPU and Memmory utlizations. 

`cpu-autoscaler.yaml`: This file contains the information about autoscaling based CPU Utlization. Here we have mention minimum and maximum no. of pods in the cluster. And also mention the average CPU utilization percentage of the pod. If it goes above the set value the autocaler triggerd and scale the no. of pods depending upon the metric. This file can also be generated using below command:
```
kubectl autoscale deployment node-app-deployment --cpu-percent=50 --min=10 --max=15 -o yaml > cpu-autoscaler.yaml
```
This command create a horizontalhorizontal pod autoscaler(hpa) object in the cluster and yaml file for the same to implement autoscaling based on cpu utilization. 


`memory-autoscaler.yaml`: This file contains almost same information for autoscaling based Memory Utlization except few changes. This can also contains the same information as above file like min and max no. of pods, name of the deployment and average memory utilization. The only change is kubernetes apiversion and resource type. 


Finally, we have to test our autoscaling by referring below Instructions:

1) Install siege package on your machine 
2) Open two termimal of master machine. 
3) Execute the below command in one terminal 
   ```
   kubectl get hpa --watch

4) Run the below siege command on another terminal to test autoscaling based on CPU Utilization
 
```   
   siege -q -c 5 -t 2m http://node-machine-IP:30080
```
5) Watch the second terminal carefullly, Here you can see whenever CPU utilization goes above then the 50%, it will incerase the no. of pods and when load decrease automatically no. of pods decreases. 

6) simillarly we can test it for memory utlization by refeering below instrctions:
   
   Enter inside the pod using **kubectl exec** comamnd 

   Install the stress package inside pod using following command:

   **apt-get update && apt-get install -y stress**

   Then run the below stess command 
```
   stress --vm 2 --vm-bytes 200
```
7) Switch to the second we to see autoscaling based on memory utlization at 60%.








   

