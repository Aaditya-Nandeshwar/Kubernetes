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
   
7) Once you logged into AWS, build the docker image and push it to AWS ECR private registry using following commands:

Retrieve an authentication token and authenticate your Docker client to your registry.
Use the AWS CLI:
```
aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com
```
Build your Docker image using the following command. You can skip this step if your image is already built:
```
docker build -t nodejs-test .
```
After the build completes, tag your image so you can push the image to this repository:
```
docker tag nodejs-test:latest ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/nodejs-test:latest
```
Run the following command to push this image to your newly created AWS repository:
```
docker push ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/nodejs-test:latest
```
8) The above steps help you to build and push your images on AWS ECR. Each time when you make any new change in the code, first you need to build your image and then push that image to a private registry like AWS ECR.


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

The above section is the general section to create any Kubernetes object. It contains the version of Kubernetes API for specific object `apps/v1`, object type `Deployment`, and name of the object along with its labels.
 
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
            cpu: "300m"
            memory: "600Mi"
          requests:
            cpu: "150m"
            memory: "60Mi"
        imagePullPolicy: Always
      imagePullSecrets:
      -  name: ecr-registry-keycc:paytm-assignment
```
The spec section is the main section of the Deployment object. Which contains more information than above. 

1) `replicas`: This is used to ensure the desired no. of pods available in the cluster after deployment. 
   
2) `selector`: It is used to select appropriate pod for deployment by referring the labels. 
   
3) `template`: This helps to create pods and containers by referring to various parameters and requirements. 
   
This section contains the name of the pods, name of the containers, name of the images, name of the secrets to pulling images from a private registry, the port where the application listens, and resources section to allocate memory and CPU from the host machine.   

I am assuming that you are aware of some of these like pods, containers, ports, and images. Let me explain to you the remaining ones which are not commonly used i.e resources and secrets. 

`resources`: As, I mentioned above this section used to allocate system resources to Kubernetes pods. Here, I am allocating CPU and Memory. The reason behind to allocate these resources to pods is, to autoscale the application pods based on CPU and Memory utilization. I have mentioned in the above limits=300m and requests=150m. This means I am allocating 0.3 Millicore CPU from my worker node. So pods can't consume more than this limit. And, I am requesting a 0.15 Millicore/pod CPU to run my application pods from this allocated limit.  

The same thing applies to memory also. But, here unit will change as `Mebibyte`(Mi) and allocating 600Mi limits for the memory from the system and requested 60Mi memory per pod used by all running pods in the deployment.

`imagePullSecrets`:This term contains the secret which can be used to pull image from private docker registry like AWS ECR in our case. Secrets are generally containing sensitive data like username, password, environment variables, etc. We used a secret to store our private registry name, username, and password for login. 

If you use the default private docker registry (docker hub), then it will be easy to manage secrets. But this is not the case with the AWS ECR (Elastic container registry). Because AWS ECR will change the authentication token in every 12 hours. It means your secret will expire after 12 hours. And you can't be used that secret anymore. To resolve this issue, I have created one shell script which can be used to rotate your secret before it expires. Please refer to the following details:

`secret-rotation-cron.sh`: This file contains a small shell script, which helps to pull the latest authentication credentials/token from the ECR registry, delete the old secret, and create the new one. 

Let me explain few lines,

```
aws ecr get-login >>  /home/ubuntu/ecr-login.sh
```
In this command, we are pulling the latest credentials from the AWS ECR registry and storing it in another `ecr-login.sh` file. This `ecr-login.sh`  used to store and rotate these credentials.
```
kubectl delete secret ecr-registry-key >> cron-output.txt
```
Using this command, we are deleting the old secret and storing the output of this command in `cron-output.txt`.  This will give assurance  
the old secret has been deleted successfully.  
```
kubectl create secret generic ecr-registry-key --from-file=.dockerconfigjson=.docker/config.json --type=kubernetes.io/dockerconfigjson >> cron-output.txt
```
The above command creates a secret from `.docker/config.json`. Because each time, when we generate login token using `aws ecr get-login` command by default its stores in `.docker/config.json` file, It is the default configuration file of docker. 

Set the cron job for your `secret-rotaion-cron.sh` file using below cron 
```
0 */10 * * *      /home/ubuntu/k8s-deployment/secret-rotaion-cron.sh
```
This cron helps to rotate secret in every 10 hours. 


After, Performing these many steps we are ready to deploy our application.  

```
kuebectl create -f nodejs-deployment.yaml
```
This command will create the deployment on the Kubernetes cluster and deploy the no. of pods mentioned in the `nodejs-deployment.yaml` file. 
To verify the deployment execute below commands:
```
kubectl get deployments 
```
It will list the no. of deployments available on the cluster under default namespace. 
```
kubectl get pods
```
It will list the no. of pods available on the cluster under default namespace. 

After deployment, we need to expose the application to the worker node.
for this, we have created one `nodejs-service.yaml` file, which will create a new Kubernetes object service. This service object help to expose the application on the worker node.   

In `nodejs-service.yaml` the file we have mention port of the container where the application listing on, Service port which is similar to container port and helps to receive incoming traffic from NodePort and forward traffic towards the container and NodePort which receives the incoming traffic. 

As per the Kubernetes documentation, Kubernetes nodes can serve the applications between port ranges `30000` to `32767`. Hence, we have used the `30080` port for exposing our application on the worker node.  

We have also configured one AWS classic load balancer for exposing the application on port `3000` instead of serving from Kubernetes default port range. We have opened port `3000` on the classic load balancer security group. Using this we can browse or application on port `3000` form any location.  

## Horizontal Pod Autoscaling  

In this section, we will discuss how to autoscale our pods based on different metrics. Kubernetes autoscale the pods by referring to the metrics of the pod and these metrics can be found with the help of metrics-server. 
That means we need one metrics server which can monitor the health of our Kubernetes resources and help to autoscale our applications. 

To achieve the autoscaling in our current deployment we have deployed default Kubernetes metrics server with the help of below metrics server YAML file.

`metrics-server.yaml`: This file contains all the necessary details to deploy the metric server. It also includes the various Kubernetes objects which can be used by metric server object to monitor pods and nodes. This includes objects like cluster role, cluster role binding, service, and API versions. 

To deploy the metric server used the below command:
```
kubectl create -f metrics-server.yaml
```
By default, it will deploy under Kubernetes default namespace which is a Kube-system. You can also find metric server pod by executing below commands:
```
kubectl -n kube-system get pods
```
It will list the no. of pods present under default namespace. 

The metric server also creates some other Kubernetes objects which you can find by executing below command. 
```
kubectl get all --all-namespaces
```
This will list all Kubernetes objects under all namespaces. 

To find the metrics of the cluster use below commands:

`kubectl top nodes`: It will list out the node, and shows the CPU and Memory utiliztion of nodes. 


`kubectl top pods`: It will list out the Pods, and shows the CPU and Memory utilization of pods.

After deployment and validation of the metrics server, we will implement autoscaling in our Kubernetes cluster. 

To implement horizontal pod autoscaling, we need to create a horizontal pod autoscaler object which can be associate with our deployment. For this, we have created two different YAML files to perform autoscaling based on CPU and Memory utilization.  

`cpu-autoscaler.yaml`: This file contains information about autoscaling based CPU Utilization. Here we have mention minimum and maximum no. of pods in the cluster. And also mention the average CPU utilization percentage of the pod. If it goes above the set value the autoscaler triggered and scales the no. of pods depending on the metric. This file can also be generated using the below command:
```
kubectl autoscale deployment node-app-deployment --cpu-percent=50 --min=10 --max=15 -o yaml > cpu-autoscaler.yaml
```
This command creates a horizontal pod autoscaler(hpa) object in the cluster and YAML file for the same to implement autoscaling based on CPU utilization.  


`memory-autoscaler.yaml`: This file contains the almost same information for autoscaling based Memory Utilization except for few changes. This can also contain the same information as an above file like min and max no. of pods, name of the deployment, and average memory utilization. The only change is the Kubernetes API version and resource type. 


Finally, we have to test our autoscaling by referring below Instructions:

1) Install siege package on your machine 
2) Open two terminal of master machine. 
3) Execute the below command in one terminal 

   ```
   kubectl get hpa --watch

4) Run the below siege command on another terminal to test autoscaling based on CPU Utilization
 
```   
   siege -q -c 5 -t 2m http://node-machine-IP:30080
```
5) Watch the second terminal carefully, Here you can see whenever CPU utilization goes above 50%, it will scale up the no. of pods, and when the load decreases below 50%, then pods will automatically scale down.  

6) similarly, we can test it for memory utilization by referring below instructions:
   
   Enter inside the pod using `kubectl` `exec` command 

   Install the stress package inside pod using the following command:
```
   apt-get update && apt-get install -y stress
```
   Then run the below stess command 
```
   stress --vm 2 --vm-bytes 200
```
7) Switch to the second terminal to see autoscaling based on memory utlization at 60%.








   

