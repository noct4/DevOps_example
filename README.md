# DevOps application example

By Bailey Ivancic

## Quick note: 
I was originally planning on deploying the dockerised React app on EKS, however I ran into complications regarding the LoadBalancer sitting in front of the service and as such couldn't hit the deployed app externally. For the sake of time, I changed my deployment strategy to use ECS for the deployment of the container instead of EKS, however the initial steps of creating/dockerising/pushing to registry is the same. 

I have still included my original write-up foe which I was planning on deploying with EKS, and the reflection questions at the end are in terms of this EKS deployment as well. I have also included a breakdown of how the ECS deployment works which is where my final product is accessible. 


## This is a quick example to deploy a working React app to a Managed Kubernetes service. In this example, EKS will be used as the Kubernetes provider of choice. There will be a few required components to the example, which are:

- Amazon account (To manage the EKS and ECR)
- ECR repository (To store the image of the application created)
- EKS cluster
- EC2 for managing the EKS cluster
- Node + React (installed locally for development)
- Docker (Installed locally for containerizing and pushing to the repository)
- Helm (Installed on the EC2 machine to deploy chart to EKS cluster)
- AWS CLI


## Steps for installation
### Common steps
1. Create applicatiom.
2. Dockerise container.
3. Push container to ECR.

### For EKS deployment
4. Create EKS cluster.
5. Enable the ServiceAccount and IAM providers
6. Create the ClusterRole and ClusterRoleBinding
7. Create yaml file for React Deployment + NodePort
8. Create the IAM policy and ServiceAccount
9. Deploy the AWS Ingress Controller (For ALB)
10. Deploy React and NodePort service
11. Test load balancer to see if the service is accessible

### For ECS deployment
4. Create an ECS cluster
5. Create a task definition
6. Create a service
7. Test public IP to see if the service is accessible

---

# Common steps

## 1. Create Application
For the application, we will be quickly creating a react application to build and serve on our cluster. This will be pacxkaged using docker and then pushed to the ECR. 
1. If it is not yet installed, install Node on your local machine.
2. To build the react app, runt he following commands:

```bash
npx create-react-app my-app
cd my-app
npm start
```
- This should start up the app in your default browser, showing you the application has been correctly installed. 

3. We will just quickly modify the app to meet the requirements of the challenge. Inside ```App.js``` within the ```src``` folder, I changed the boilerplate text to say "Hi there! Welcome to Bailey's example React site!". The rest of the page is fine for this challenge, and so it was left be. 


## 2. Dockerise container
To dockerise the container, we will create a Dockerfile that will package the app into a container, which we can then ship and deploy.
1. Create the Dockerfile inside the root directory of the react app. This file should be called ```Dockerfile``` and will contain the following:
```Dockerfile
FROM nginx:stable-alpine

COPY build/ /usr/share/nginx/html 
```

2. Add a ```.dockerignore``` file containing the following (to minimise the size of the image) containing the following:
```.dockerignore
node_modules
.dockerignore
Dockerfile
```

3. Package the built react app into a Docker container:
```bash
docker build .
```
- This will go through the build process and, once done, will output the docker image hash

4. As a sanity check, running a ```docker image ls``` should show the docker images that have been created locally. You should see the docker image freshly built on top of the list, with the same hash that was given after the docker build command.

5. **As an aside, when testing the Docker container I found that I was getting strange errors and the app wasn't running. After some investigation I found that my Apple-Silicon-based Macbook was building the images as ARm64 images, which meant that when these were deployed to the Amazon services I was using, they were incompatible. This was fixed by running the docker build command with a ```--platform linux/arm64``` flag**

## 3. Push the container to ECR
Now that we have our dockerised application, we need to push it to a container registry, so that when we deploy the application in our EKS/ECS it can pull the image from somewhere. The container refgistry we will be using will be ECR, which is the AWS container registry solution.

1. Create the ECR repository through the AWS console. This process will not be detailed here, but there are plenty of guides available online if help is needed. The repository can be created as a public repo for this exercise, although in a production environment this would be a private repository with additional safeguards.

2. We now need to push the local docker image we have created into the ECR. This step will assume that the AWS CLI has been installed and AWS has been authenticated with Docker. If this has not been done, follow this documentation from AWS to get started: https://docs.aws.amazon.com/AmazonECR/latest/userguide/getting-started-cli.html. 

3. Once Docker has been authenticated, create a repository in the ECR through the web console. 

4. Once this is created, follow the commands given in the 'View push commands' button which is found when clicking in the repository that has just been made. These commands are doing the following actions:
    - Tagging the created image with ecr_repo_name:tag
    - Pushing the tagged image to the newly-created ecr repo

## **For EKS deployment**

## 4. Create EKS cluster
We will now create the EKS cluster that will host our sample application. While this is a bit overkill for running this simple application, it allows us to picture what a larger-scale application deployment would work, since Kubernetes provides a solution for scaling and deployment of large microservice applications.

1. Before we create an EKS cluster, we need to create an IAM role for the cluster to use.
- Through the IAM console, click on Create Role
- The 'Trusted entity type' will be 'AWS service', the 'Use case' will be 'EKS - Cluster', and everything else will be left as default. Give the role a name like 'myEKSClusterRole'.

2. Create an EKS cluster using the EKS console on the AWS site. Give it a name, such as 'DevOps_example_cluster', leave the K8s version as the default and select the cluster role you just created in the earlier step. In terms of the Cluster endpoint access, we will be choosing 'Public and private', allowing us to hit the endpoint cluster while keeping the internal traffic private. In a production environment, this would be changed to likely a Private access only with additional controls. Leave logging off for this exercise. On the review page, click 'Create'.

3. Next we need to configure the Amazon VPC CNI plugin for use with a serviceaccount in our cluster. Follow the documentation given here: https://docs.aws.amazon.com/eks/latest/userguide/cni-iam-role.html.

4. Now that the EKS cluster is created, we need to add a NodeGroup into the cluster to allow it compute resources (to deploy our application onto). A prerequisite of this is a role for the NodeGroup to assume. The following guide shows the steps to create the role: https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html. I have called the role 'myEKSNodeRole'.

5. With these roles created and the cluster set up, we need to create a NodeGroup within the cluster. Within the 'Compute' tab of the cluster console, click on 'Add node group'. Give it a name, in this case my name is 'NodeGroup1', and select the NodeGroup which was just created. On the next page on config, I will be using a t2.micro instance, since the application is just to test the DevOps deployment, and the scaling will just be set to 1. Everything else will be left the same. The NodeGroup will take a little bit of time to create, but if done correctly will say 'Active' as the status.

6. Now that the cluster is created, follow the AWS documentation here: https://aws.amazon.com/premiumsupport/knowledge-center/eks-cluster-connection/ to connect your local kubectl tool to the cluster. At the end, if connected correctly, running ```kubectl get svc``` should show the following:
```
NAME         TYPE        CLUSTER-IP          EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   <YOUR_CLUSTER_IP>   <none>        443/TCP   5d4h
```

7. **As an aside, when trying to debug my config errors I used a yaml-defined workflow for creating the EKS cluster, which is found in ```yaml/cluster.yaml```.**


## 5. Enable the ServiceAccount and IAM providers
This was done by following the AWS documentation laid out on https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html.

## 6. Create the ClusterRole and ClusterRoleBinding
These were created by applying the ```rbac-role.yaml``` file. 

## 7. Create yaml file for React Deployment + NodePort
The first yaml we need to create is the yaml file which will deploy our app onto the cluster. This contains the image we want to deploy as well as specifications for how to deploy it. The entire deployment details can be set with this yaml, however for this challenge only a basic deployment yaml is needed.

The second yaml will create a NodePort, a service which will be used to expose a service within our cluster for the LoadBalancer to direct traffic to.  

These can both be put into one file which will specify the whole deployment.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bailey-react-deployment
spec:
  selector:
    matchLabels:
      app: bailey-react-app
  replicas: 1
  template:
    metadata: 
      labels:
        app: bailey-react-app
    spec:
      containers:
        - name: bailey-react-app
          image: public.ecr.aws/g1f2n3m1/react_example_app:latest
          ports:
            - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: bailey-react-app-nodeport
spec:
  type: NodePort
  selector:
    app: bailey-react-app
  ports:
    - port: 3000
      targetPort: 3000
```

## 8. Create the IAM policy and ServiceAccount
The IAM policy was created through reference of the AWS documentation, specifically with the config found in ```iam_policy.json```.

TYhe ServiceAccount was applied through the yaml found on ```sa.yaml```

## 9. Deploy the AWS Ingress Controller (For ALB)
The ingress controller was deployed through following the AWS documentation at https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html. A copy of the specification is found in ```v2_4_1_full.yaml```.

## 10. Deploy React and NodePort service
Since we already have the yaml defines for these components, we just need to apply the specification with ```kubectl apply -f deployment.yaml```. The output should be as follows:
```
deployment.apps/bailey-react-deployment created
service/bailey-react-app-nodeport created
```

It might take some time, but you should eventually see the services ready by running ```kubectl get deploy``` and ```kubectl get svc```.

## 11. Test load balancer to see if the service is accessible
By describing the load balancer, it should provide an external address which the service will be available at.
In my case, i couldn't figure out why the external IP address I was given was failing, and so I decided to change my deployment strategy.


## **For ECS deployment**
### 4. Create an ECS cluster
- Through the AWS console, go into the ECS service section and select 'Create Cluster'. 
- We will be creating a Networking Only cluster for this challenge, so once that is selected along with the desired VPC and a name given to the cluster, click 'Create'.

### 5. Create a task definition
- Using the left sidebar, click 'Task Definitions'.
- Aside from giving the definition a name, the most important thing to do in this step is to provide the definition with the ECR image details so that it can pull and deploy the app. These are given in the rrespective text fields during the creation process.

### 6. Create a service
- Once the Task Definition is complete, we will use this to create a Service
- Inside the 'Services' tab click 'Create'.
- We will be using Fargate as the launch type, with the Cluster being the one that was created earlier and '1' for the 'Number of tasks' field. Configure with a desired VPC and subnet, and click through the review page. The finalised deployment should show within the Task tab.

### 7. Test public IP to see if the service is accessible
The public address can be found by clicking into the task and looking at the "Public IP" field inside the "Network" section. If all steps were done right, hitting ```http://<IP>``` will return the customised React splash screen!

---
# Other documentation
## Switch from EKS to ECS
Unfortnuately, due to complications with the LoadBalancer I made the switch to an ECS deployment over an EKS one. While the outcome for this challenge is the same, the environments are suited to differrent purposes, and depending on how the app planned to be developed in the future might mean that the EKS route is the best way to go.

This last-minute switch does highlight the benefit of this kind of workflow however, as I was able to use my already dockerised react image and simply change the method of deployment, leveraging the platform-agnostic and ephemeral nature of containerised apps.

## Possible future plans for the website
There is a lot that could be done with the website due to the way it has been created.

I decided to make the app using React. This is definitely not needed for the simple requirements of the challenge, but this deployment shows that even a sizeable react application can be deployed using this workflow. As a result, a large react-based app with actual business logic can be created like this, as well as other microservice components like a backend and database depending on solution needs. 

By deploying the application through Kubernetes, we have created a solution which utilises the K8s orchestration engine to deploy and manage our images. This means that if we decided to build this app further in the future and feature different components within the solution, with some configuration changes we would be able to utilise the benefits of the K8s system to scale and provide redundancy to the app. 

## Alternative solutions
Since the challenge specifications are quite broad, the challenge can be complated a larrge number of ways. Essenttially, this was to just create a static site utilising AWS or GCP technologies. This can be done a numbe rof ways, such as AWS Amplify, S3 or Lightsail. If the challenge wasn't limited to AWS or GCP technologies, something like Heroko could even be used, or a simpler solution still. For the purposes of this challenge there wouldn't be anything wrong with this solution, however wit would create a barrier if the app was gouing to be developed further and more complexity was needed. 

I decided to try and host the site using a Kubernetes deployment flow to potentially imagine the challenge as the MVP for a much larger project, which would eventually branch out and become a (hypothetical) react product. Through this K8s environment, and subsequent tools used in deployment, the app would be built in a containerised, microservice environment, which is indicative of a modern application adhering to the 12-factor app principles. 

## Requirements to make a production grade website
### Automation
As shown from the guide, this is all currently a manual process to deploy the application. In a production-grade environment, this setup could potentially be automated with tools such as Ansible and CloudFormation, which would both save time/manpower in setting up the resources but also allow for automated deployment of differemnt environments, for example dev and prod. This also continues the idea that environments should be ephemeral and temporary, and should have the capacity to be re-created at any time.

### Security
A few liberties were taken due to this just being a testing challenge, but security would need to be increased within a production environment. Not only would images not be publically available, but other security practices such as AWS Security Groups, IAM and K8s roles would need to be developed to ensure only proper access to the cluster and resources is allowed.

### CI/CD
In a production environment, there would ideally be some sort of CI/CD pipeline, which would be tied into the automation aspect. Depending on team/project needs, this could feature automated testing, environment variable management, management of secrets and much more. This would also allow for rapid builds on predetermined events and a faster, less manual pipeline. 