# Alltrails DevOps Interview Assignment

## Overview
The objective of this repository is to define the infrastructure needed to create a reverse proxy to route traffic to both a pre-existing Rails app and a new Navigation service.

### Assumptions
* The Rails app is already exposed to internal cluster traffic via a service resource.
* An ingress controller has already been deployed to the cluster.
* Metrics-server has already been deployed to the cluster.
* The domain name attached to our cluster is `alltrails.com`
* There is a valid TLS cert and key that can be loaded into the `tls-secret.yaml` file for secure traffic.

### Overview of the Nginx proxy:

* All components are packaged in a helm chart for easy deployments and templating ability.
* The Nginx configuration is defined in a configmap, which is then mounted into the nginx pod.
* The Nginx reverse proxy deployment is exposed to the cluster with a service.
* An ingress object is created to expose the Nginx reverse proxy service to traffic from outside the cluster. TLS is terminated here by referencing the tls-secret object.
* Traffic will enter the cluster through the ingress, be routed to the Nginx proxy container, which will then route the request to either the Rails app or Navigation app depending on the path of the request URL.

### Overview of the Navigation service:

* All components are packaged in a helm chart for easy deployments and templating ability.
* A deployment is created for the application.
* Each pod contains an init container and the main app container.
* The init container downloads the desired OSM file from an external source and mounts it into a volume that the app container can then use at startup. This way, we do not have to store the OSM files on the host machine, giving us the ability to run it in a production cluster as well as a local dev cluster.
* The app container runs the openrouteservice image, using the OSM file that was mounted into the shared pod volume.
* The deployment is exposed as a service, so internal traffic from the nginx proxy can reach it.
* Horizontal Pod Autoscaling can be enabled/disabled in the values.yaml file. This requires metrics-server to be running in the cluster.

## Navigation service API

We are directly using the openrouteservice image for our service, since there is no need to build our own. The only configuration change considered in this scenario is the possibility of using a different OSM file. This can be mounted into the container and does not need to be defined at build time.

Usage:
* External traffic would send a request to https://alltrails.com/navigation/ors/v2/directions/driving-car?start=XXXXX,YYYYY&end=XXXXX,YYYYY
* Replacing the two sets of coordinates in the above URL to the starting and ending locations.
* The API would then return a GeoJSON blob which contains the directions.

## Deploying to Production

Since the nginx proxy and Navigation service are both packaged in helm charts, it makes deploying to production very easy.
The simple, manual way of doing it is to install helm in the cluster, and then deploy the app via a helm command like so:

```
helm install navigation-service navigation-service/ --values navigation-service/values.yaml
helm install nginx-proxy nginx-proxy/ --values nginx-proxy/values.yaml
```

An alternative approach that is more scalable is to use a CD tool such as ArgoCD to deploy the two apps.
You would create an ArgoCD app resource, and configure its source to be the desired helm chart. ArgoCD takes care of the rest and you would not have to manually upgrade the chart each time you make a change to one of the resources.

## Testing locally

This would require a local or dev cluster to deploy the services.

Two possible ways of testing locally:
1. Set the nginx-proxy service to be of type NodePort. This will allocate a port on your cluster that you can reach externally from the dev machine. You can configure a test script that sends several requests to your cluster on this port, and you can see how the traffic hits the Rails or Navigation apps.
2. Adjust your machine's localhosts file to have `alltrails.com` point to your cluster's IP. This should test requests you send to `alltrails.com` be routed to your local cluster instead through the ingress. This can be be complicated by the TLS cert however, and may cause a need for additional configuration.

## Answers to Questions

1. How do the new resources scale?

The Navigation Service can be enabled for Horizontal Pod Autoscaling in the helm chart's values.yaml file. This feature will watch the CPU usage of each pod in the deployment. If the average CPU usage exceeds the threshold of 70% capacity, a new pod will start up to better handle the load. This eliminates the need for manually increasing the replica count.

2. How would you adapt the app to use a much larger OSM file, such as the one for Northern CA (451MB)?

Using helm templating, you are able to update the source, file name, and file extension of the OSM file in the helm chart's values.yaml file. If you wanted to adjust the OSM file, you would simply update those values and redeploy/upgrade the chart. Similarly, the navigation service pod memory limits and requests are also done with helm templating. If you know the OSM file will be quite large, increase these values in the values.yaml file and redeploy/upgrade the chart so the pod has enough resources to function properly.

3. How does your infrastructure handle security?

The nginx-proxy ingress resource requires the incoming traffic to be HTTPS. A TLS key/secret is configured here so that TLS is terminated at the ingress, and valid traffic can move around in the cluster in an HTTP scheme. This setup ensures that outside traffic must pass through the validating ingress before hitting any of our actual applications.

## Challenges/Improvements

* Could have built a more robust wrapper around the openrouteservice image to have a frontend interface for users. I do not have much experience with this and spent more time on the other portions of the assignment. 
* While the nginx-proxy works, it is an uneccessary step. Instead, we can install an ingress-controller and define ingress resource(s) that have the capabilities to do path-based routing to different services in our cluster. This is a more straightforward approach.