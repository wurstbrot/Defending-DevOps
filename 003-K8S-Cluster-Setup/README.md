# Kubernetes Cluster Setup

## Deploying Your App to Kubernetes
(!)This requires that you have GKE and Cloud Shell configured correctly. Please go back to `001-Lab-Setup` if you are not set up.

### Task 1: Getting to Know Your Cluster
1. `kubectl` is the command line utility that we will use to interact with our Kubernetes cluster. The first task is to view the Pods that are running on our cluster with an out-of-the-box installation. Run the following command in you terminal:
```
kubectl get pods
``` 

2. As you can see no pods are running. This is because we are dropped into the `default` namespace and the `default` namespace has nothing deployed to it. Try running the same command with the following argument. This will list the pods used by the Kubernetes system itself:
```
kubectl get pods --all-namespaces
```

3. Let's take a look at a new command called describe. Run the following command to inspect the details about our cluster:
```
kubectl describe node
```

4. Use the `describe` command to describe one of the pods running in the kube-system namespace.

Hint: Check out the official [kubectl cheatsheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) for more useful tips.

### Task 2: Running our Application in GKE
1. Launch the application by creating a Kubernetes Deployment using the following command (this pulls down an image from Docker Hub so it may take a few minutes):

Note: DO NOT CHANGE IMAGE NAME TO YOUR OWN - THIS IS AN IMAGE ON DOCKER HUB

```
kubectl run link-unshorten --image=jmbmxer/link-unshorten:0.1 --port=8080
```

2. Take a look at your running Pods and make sure the container has been created successfully:
```
kubectl get pods
```

3. You will also see that this command created a Deployment called link-unshorten. Run this command to show the Deployments running on our cluster:
```
kubectl get deployments
```
You will notice some information regarding our new deployment in the output.

4. Now, inspect the details of this deployment using `describe`:
```
kubectl describe deployment link-unshorten
```

5. Run the following command to view more info about the deployment you just created. We use the `-l` flag here to tell Kubernetes to only get pods with the Label of `run=link-unshorten` which is the default label given to this particular Pod:
```
kubectl get pods -l run=link-unshorten -o yaml
```

6. We can use grep to extract the IP address that was assigned to our Pod:
```
kubectl get pods -l run=link-unshorten -o yaml | grep podIP
```

7. Use kubectl to get a shell to your container:
```
kubectl exec -it <podname> /bin/bash
```
Hint: Retrieve the `<podname>` using `kubectl get pods`

8. If you `ls` in the shell you will see the golang app source code

9. We can use `curl` to hit our API from localhost once we have a shell:
```
curl localhost:8080/api/check?url=bit.ly/test
```
Exit out of the container
```
exit
```

### Task 3: Exposing your Pod to the World
There are a variety of ways to make our Pod accessible to the outside world. A Service with the type `LoadBalancer` will be used to give our Pod a stable existence and an IP we can reach from our web browser.

The `LoadBalancer` type spins up a load balancer in GCP automatically. 

1. To expose the application we create a Service with the type of LoadBalancer:
```
kubectl expose deployment link-unshorten --type=LoadBalancer

```

2. We can now see our new Service details by running the following command:
```
kubectl get svc
kubectl describe svc link-unshorten
```

3. Visit the IP address listed in the terminal in your browser (labeled as `LoadBalancer Ingress`. Don't forget to add the API endpoint path.
```
http://<EXTERNAL-IP>:8080/api/check?url=bit.ly/test
```
4. This is no way to manage a real Kubernetes cluster. Tear down your app using the following commands:
```
kubectl delete deployment link-unshorten
kubectl delete svc link-unshorten
``` 

### Task 4: "Codifying" Your Deployment
Running ad hoc commands in a terminal are no way to maintain a proper DevOps infrastructure. Kubernetes is built with "Infrastructure as Code" in mind by using manifests. Manifests can be written in JSON and YAML. We will be using YAML for all labs.

1. In the `manifests` folder of this lab you will find a few files needed to launch our API. Open them up in a the Cloud Shell text editor and take a look.

2. Go to the `manifests` directory using your terminal and use kubectl to launch the Service and the Deployment in your cluster. The `-f` flag is used to specify a manifest file:
```
kubectl create -f link-unshorten-deployment.yaml
kubectl create -f link-unshorten-service.yaml
```

3. Make sure the pods are running without error:
```
kubectl get pods
```

4. Under the hood we can see the new ReplicaSet that was created. Remember, a Deployment actually creates a ReplicaSet. Deployments provide the same replication functions via ReplicaSets and also the ability to rollout changes and roll them back if necessary. 
```
kubectl get replicaset
```

5. Check out your newly created "microservice" using the following command to extract the IP address:
```
kubectl describe svc link-unshorten-service
```

6. Similar to how we interacted with our application earlier, we use the IP from the above output and paste it into our browser.
```
http://<EXTERNAL-IP>/api/check?url=bit.ly/test
``` 

7. Deployments offer the ability to scale Pod counts simply. Open the Deployment manifest and scale the number of pods to six. Once the change has been made and saved, use the `replace` command to scale your Deployment. You can also use `apply` here to accomplish the same result. Too the moon!
```
kubectl replace -f link-unshorten-deployment.yaml
```

8. Inspect the Pods scaling. Note that others will be terminating at the same time:
 ```
 kubectl get pods 
 ```
 
 Inspect all six pods at once:
 ```
 kubectl get pods | grep -V NAME | awk '{print $1}' | xargs kubectl describe pod
 ```

9. Un-comment the redis container lines in the link-unshorten-deployment.yaml manifest to deploy a second container within our Pod. Use `kubectl replace -f link-unshorten-deployment.yaml` to commit the changes after the lines have been un-commented.

10. If you are curious about container-to-container communication within a running Pod, exec into the Redis container using the following command. The name of the Redis container is `unshorten-redis-cache` which is declared in the link-unshorten-deployment.yaml file.
```
kubectl exec -it <PodName> -c unshorten-redis-cache /bin/bash
redis-cli ping
```

11. This Redis container has very few Linux packages installed (a good thing!) so we can go get curl using the following command. This is for demonstration purposes. It is not recommended to install ad hoc tools in running containers...remember CATTLE not PETS!
```
apt-get update && apt-get install curl
```

12. Since containers within a Pod communicate over Localhost, we are able to access our API endpoint using curl as follows:
```
curl 127.0.0.1:8080/api/check?url=bit.ly/test
```
### Task 5: Cleanup
Tear down our deployment and service as follows:
```
kubectl delete deployment link-unshorten
kubectl delete service link-unshorten-service
```
### Bonus
 A critical RCE vulnerability was just reported through a bug bounty and was fixed late into the night. Roll out a new version of the app (0.2) in your cluster to patch the vulnerability on each of your three running pods. No downtime allowed! Show the deployment history using `kubectl rollout history` 

### Bonus
The new version you just rolled out contains a critical bug! Quickly rollback the deployment to 0.1 (Yes, 0.1 is the vulnerable version, but this is just for practice!)

### Discussion Questions
1. What would be a good piece of your application or infrastructure to start breaking up into Pods within Kubernetes? 

2. What security challenges does administering a Kubernetes cluster using a tool like kubectl present? 
