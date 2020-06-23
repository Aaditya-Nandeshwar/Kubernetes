# k8s-deployment

## About

This document briefly describe how to deploy and scale nodejs application on kubernetes, using horizontal pod autoscaler running on AWS cloud.

## prerequisite

1) Create a 2 EC2 linux intances on AWS with minimum configuration of 2 vCPU and 4GB of RAM and name them as Master and Node. 
2) Create one AWS IAM user with neccessary permission, note down ACCESS_KEY and SECRET_KEY of IAM USER. This credentials we will use to login on AWS.
      
3) Create one ECR repository with name nodejs-test on AWS.  
    
4) Setup a kubernetes cluster on AWS using kubeadmin tool. Please refer the following link for reference:
   https://www.edureka.co/blog/install-kubernetes-on-ubuntu
   
   https://vitux.com/install-and-deploy-kubernetes-on-ubuntu/

   Noete: You can also refer to the kubernetes official documention. 

## Description 

1) Clone the above application from github on AWS Master EC2 instance using following command:

git clone https://github.com/Aaditya7789/k8s-deployment.git
   
    
2) Then siwtch to **k8s-deployment** directory. Here you will see all files and folder related to your deployment.
    
3) For deploying any application on the k8s you need a docker image which stored in a any private or public docker registry like  Dockerhub, AWS ECR (Elasctic container registry) or gcr (Google container registry). 
   
4) As mentioned above you need Docker image to deploy your application. This we can get by building an image from Dockerfile stored in **nodejs-app** directory. 
   
5) Before building the docker image we need to login into AWS from our terminal using aws-cli. Download the aws-cli on your terminal using following link:
   
   https://aws.amazon.com/cli/
    
6) For login use the same AWS ACCESS_KEY & SECRET_KEY generated at the time of user creation. 
   
7) Once you logged in build the docker image and push it to AWS ECR private registry using following commands:

Retrieve an authentication token and authenticate your Docker client to your registry.
Use the AWS CLI:

aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin ACCOUNT_ID.dkr.ecr.us-east-2.amazonaws.com

Build your Docker image using the following command. You can skip this step if your image is already built:

docker build -t nodejs-test .

After the build completes, tag your image so you can push the image to this repository:

docker tag nodejs-test:latest ACCOUNT_ID.dkr.ecr.us-east-2.amazonaws.com/nodejs-test:latest

Run the following command to push this image to your newly created AWS repository:

docker push ACCOUNT_ID.dkr.ecr.us-east-2.amazonaws.com/nodejs-test:latest







   
   
   





   

