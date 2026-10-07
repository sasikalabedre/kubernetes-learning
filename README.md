Project - Dev and Prod on GKE
What I built

One private GKE cluster with two namespaces running the same app (nginxdemos/hello):

	dev	prod
Replicas	1	3
Service type	ClusterIP (internal only)	LoadBalancer (public IP)
Setup
GKE Standard cluster, zone asia-south1-a, 2 nodes (e2-medium)
Private nodes (no public IPs on the nodes)
Cloud NAT so the private nodes can pull images from the internet
kubectl connected with gcloud container clusters get-credentials
Files in gke-project/
deployment.yaml (same file used for both dev and prod)
service-dev.yaml (ClusterIP)
service-prod.yaml (LoadBalancer)
What I tested
Load balancing: sent 30 requests to the public IP. The 3 prod Pods answered 11, 7 and 12 times. With only 10 requests the split was uneven (5, 4, 1), so more requests give a more even spread.
Self-healing: deleted one prod Pod. The ReplicaSet created a new Pod with a new name to get back to 3, and the app kept working.
Rolling update: changed the image with kubectl set image and watched the Pods get replaced one by one with no downtime.
Bad release: deployed an image tag that does not exist. The new Pod went to ImagePullBackOff, but the old Pods kept serving users.
Rollback: kubectl rollout undo brought back the previous working version.
Problems I faced and how I fixed them
Problem	Cause	Fix
gke-gcloud-auth-plugin: command not found	gcloud installed with snap does not include the plugin	Added Google's apt repository and installed google-cloud-cli-gke-gcloud-auth-plugin
Cluster creation failed with compute.vmExternalIpAccess violated	The project has a policy that blocks public IPs on VMs, and GKE nodes get public IPs by default	Created a private cluster (--enable-private-nodes) and added Cloud NAT
kubectl get nodes gave an i/o timeout	The control plane only accepted allowed IPs, and my IP was not on the list	Added my public IP with --master-authorized-networks <my-ip>/32
grep "Server name" printed nothing	The text was not matched in the page HTML	Searched for the Pod name pattern instead: grep -o 'hello-app-[a-z0-9]*-[a-z0-9]*'
<your-ip> command failed	I typed the placeholder literally, and bash read < as a file redirect	Replaced the placeholder with the real IP
Main commands used
bash
# Create private cluster with Cloud NAT
gcloud compute routers create nat-router --network default --region asia-south1
gcloud compute routers nats create nat-config --router nat-router --region asia-south1 \
  --auto-allocate-nat-external-ips --nat-all-subnet-ip-ranges
gcloud container clusters create demo-cluster --zone asia-south1-a --num-nodes 2 \
  --machine-type e2-medium --disk-size 30 --enable-ip-alias \
  --enable-private-nodes --master-ipv4-cidr 172.16.0.0/28
gcloud container clusters get-credentials demo-cluster --zone asia-south1-a

# Deploy
kubectl create namespace dev
kubectl create namespace prod
kubectl apply -f deployment.yaml -n dev
kubectl apply -f service-dev.yaml -n dev
kubectl apply -f deployment.yaml -n prod
kubectl apply -f service-prod.yaml -n prod
kubectl scale deployment hello-app --replicas=3 -n prod

# Release and rollback
kubectl set image deployment/hello-app hello=nginxdemos/hello:plain-text -n prod
kubectl rollout status deployment/hello-app -n prod
kubectl rollout undo deployment/hello-app -n prod
kubectl rollout history deployment/hello-app -n prod
Cleanup

To avoid charges, I deleted everything after the project, in this order:

Namespaces (this releases the load balancer)
The GKE cluster
Cloud NAT and Cloud Router
What I learned
A Deployment keeps the desired number of Pods, and a Service gives them one stable address.
A bad release does not cause downtime, because Kubernetes only removes old Pods after the new ones are ready.
Namespaces let me run dev and prod in one cluster using the same manifest.
Cloud policies can block a default setup, and a private cluster with Cloud NAT is a safe way around it.
Always check the current context before running commands.
