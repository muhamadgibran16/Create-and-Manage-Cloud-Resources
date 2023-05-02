# Create-and-Manage-Cloud-Resources
Challenge Lab Google Cloud Computing Foundations (Google Skillboost)

============================================================================================
Task 1. Create a project jumphost instance
You will use this instance to perform maintenance for the project.

Requirements:

Name the instance Instance name.
Use an f1-micro machine type.
Use the default image type (Debian Linux).

---------------------------------------------------------------------------------------------

gcloud compute instances create nucleus-jumphost-878 \
          --network nucleus-vpc \
          --zone us-west3-c  \
          --machine-type f1-micro  \
          --image-family debian-10  \
          --image-project debian-cloud
          
          
============================================================================================

Task 2. Create a Kubernetes service cluster

Requirements:

Create a zonal cluster using <filled in at lab start>.
Use the Docker container hello-app (gcr.io/google-samples/hello-app:2.0) as a placeholder; the team will replace the container with their own work later.
Expose the app on port App port number.

---------------------------------------------------------------------------------------------

gcloud container clusters create nucleus-backend \
          --num-nodes 1 \
          --network nucleus-vpc \
          --zone us-west3-c
gcloud container clusters get-credentials nucleus-backend \
          --zone us-west3-c

kubectl create deployment hello-server \
          --image=gcr.io/google-samples/hello-app:2.0

kubectl expose deployment hello-server \
          --type=LoadBalancer \
          --port 8080

============================================================================================


Task 3. Set up an HTTP load balancer
You will serve the site via nginx web servers, but you want to ensure that the environment is fault-tolerant. Create an HTTP load balancer with a managed instance group of 2 nginx web servers. Use the following code to configure the web servers; the team will replace this with their own configuration later.

Requirements:

Create an instance template.
Create a target pool.
Create a managed instance group.
Create a firewall rule named as Firewall rule to allow traffic (80/tcp).
Create a health check.
Create a backend service, and attach the managed instance group with named port (http:80).
Create a URL map, and target the HTTP proxy to route requests to your URL map.
Create a forwarding rule.

---------------------------------------------------------------------------------------------

cat << EOF > startup.sh
#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOF

gcloud compute instance-templates create web-server-template \
          --metadata-from-file startup-script=startup.sh \
          --network nucleus-vpc \
          --machine-type g1-small \
          --region us-west3

gcloud compute instance-groups managed create web-server-group \
          --base-instance-name web-server \
          --size 2 \
          --template web-server-template \
          --region us-west3

gcloud compute firewall-rules create permit-tcp-rule-378 \
          --allow tcp:80 \
          --network nucleus-vpc
gcloud compute http-health-checks create http-basic-check
gcloud compute instance-groups managed \
          set-named-ports web-server-group \
          --named-ports http:80 \
          --region us-west3

gcloud compute backend-services create web-server-backend \
          --protocol HTTP \
          --http-health-checks http-basic-check \
          --global
gcloud compute backend-services add-backend web-server-backend \
          --instance-group web-server-group \
          --instance-group-region us-west3 \
          --global

gcloud compute url-maps create web-server-map \
          --default-service web-server-backend
gcloud compute target-http-proxies create http-lb-proxy \
          --url-map web-server-map

gcloud compute forwarding-rules create http-content-rule \
        --global \
        --target-http-proxy http-lb-proxy \
        --ports 80
gcloud compute forwarding-rules list

============================================================================================
