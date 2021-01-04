---
published: true
---
# Enter Kubernetes, Docker, Flask, and EKS.

That's a lot of buzzwords, but combined they can powerfully orchestate and deploy an app(a Flask app will be used in this example). [Kubernetes](https://kubernetes.io/) is designed to give you less headaches, not add complexity.  

I recently switched jobs and went from using AWS about 20% to 100% of the time, which has driven me to doing more personal projects on AWS. Here's how to deploy a Flask app to AWS EKS using Kubernetes and Docker.


## eksctl.io

Outside of the usual using EKS with Terraform, I stumbled across [eksctl](https://eksctl.io), which appeared to be the quickest way to deploy a working kubernetes stack. Install eksctl to get started. **This post assumes you already have working AWS credentials**. Eksctl works off of yaml files to build clusters. I found this preferable over running eksctl create cluster - 

```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: testcluster
  region: us-west-2

nodeGroups:
  - name: nodegroup-1
    instanceType: t2.small
    desiredCapacity: 3
    privateNetworking: true
    iam:
      withAddonPolicies:
        autoScaler: true
        albIngress: true
        imageBuilder: true
```

** You can find all the files used in the example repo to copy/paste [here.](https://github.com/sadminriley/PulpFictionQuoteAPI/tree/master/deploy)

You may want to change the clustername, region, nodegroup name, or instance type/capacity.

Save this as cluster.yaml and run the following - 

```
eksctl create cluster --config-file=cluster.yaml
```

Eksctl will start running and may take sometime. For the above cluster.yaml, it took my runs about 10-15 minutes. You can always check the progress of the stack with the following - 

```
eksctl utils describe-stacks --region=us-west-2 --cluster=testcluster
```

It should autocreate the .kube/config file but if it doesn't you can always run - 

```
aws eks --region us-west-2 update-kubeconfig --name testcluster
```

Assuming you have [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) installed, you should be able to **kubectl get nodes** and see some output similar to the following with nodes in the Ready status - 

```
NAME                                           STATUS   ROLES    AGE    VERSION
ip-192-168-111-24.us-west-2.compute.internal   Ready    <none>   138m   v1.18.9-eks-d1db3c
ip-192-168-131-98.us-west-2.compute.internal   Ready    <none>   138m   v1.18.9-eks-d1db3c
ip-192-168-98-153.us-west-2.compute.internal   Ready    <none>   138m   v1.18.9-eks-d1db3c
```
Yes, that's it. Kubernetes is easy, right? *evil grin*
## Flask with Docker

I'm going to deploy [this repo](https://github.com/sadminriley/PulpFictionQuoteAPI) in this example thats already configured with a sqlite database. This repo is a very basic Flask API with minimal routes and an added healthcheck(more on healthcheck adventures later, maybe.) that utilizes Docker. 

Next, I took my dockerized repo and uploaded it to AWS ECR. You can find a tutorial on doing this here - [Docker Push ECR Image](https://docs.aws.amazon.com/AmazonECR/latest/userguide/docker-push-ecr-image.html) 


With the docker image and repository ready to go with a working Flask app, let's create a deployment and a service for it. Here I use port 5000, as I will be publically exposing it from the k8s cluster.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: appdeploy
  labels:
    app: flask
spec:
  selector:
    matchLabels:
      app: flask
  replicas: 3
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: flask
    spec:
      containers:
      - name: appdeploy
        image: public.ecr.aws/imagenamegoeshere:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 5000

---
apiVersion: v1
kind: Service
metadata:
  name: appdeploy
  labels:
    app: flask
spec:
  ports:
  - port: 5000
    protocol: TCP
    name: flask
  selector:
    app: flask
  type: LoadBalancer
```

Since the Dockerfile and running the Flask app are not port specific, we will set it above.

Save this as deploy.yaml and run **kubectl apply -f deploy.yaml**. This will create the deployment and the service(Load balancer in order to access the API). You should be able to run **kubectl get svc** to see the External IP of the Load Balancer we just created. You should also be able to see the deployment - 

```
➜ kubectl get deploy
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
appdeploy   3/3     3            3           164m
```

Check the pods to make sure they are in the Running status with **kubectl get po -n default**. 

Now, you should be able to hit the URL with our Flask api that the load balancer provided after running **kubectl get svc**. 

```
curl -X GET urlgoeshere.us-west-2.elb.amazonaws.com:5000/
"Healthy!"

curl -X GET urlgoeshere.us-west-2.elb.amazonaws.com:5000/characters
{"Available Characters":[{"name":"jules"},{"name":"vincent"},{"name":"mia"}]}

curl -X GET urlgoeshere.us-west-2.elb.amazonaws.com:5000/quote/vincent
{"vincent says...":[{"quote":"Thats a pretty fucking good milkshake. I dont know if its worth five dollars but its pretty fucking good"}]}

curl -X GET urlgoeshere.us-west-2.elb.amazonaws.com:5000/quote/jules
{"jules says...":[{"quote":"You ever read the Bible, Brett?"}]}

curl -X GET urlgoeshere.us-west-2.elb.amazonaws.com:5000/quote/mia
{"mia says...":[{"quote":"When in conversation, do you listen, or do you just wait to talk?"}]}
```

Here, we see a working, high quality Pulp Fiction quote api in action.

_...Nice_

You now have a basic, working cluster ready to deploy and modify as you please. This is a very simple surface scratch of the capabilities of Flask and the platforms/software used. I may get into more advanced kubernetes and flask usage in later posts.

- Riley

**Links** - 

[Used Example Flask API with Docker from Riley's github](https://github.com/sadminriley/PulpFictionQuoteAPI/)

[Flask API Tutorials](https://flask-restful.readthedocs.io/en/latest/)

[eksctl.io](https://eksctl.io/)

[Kubernetes](https://kubernetes.io)

[AWS EKS](https://aws.amazon.com/eks/)
