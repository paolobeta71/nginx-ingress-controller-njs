# nginx-ingress-controller-njs

This repository is based on [Fabrizio Fiorucci's repository for NGINX-AuthN-Auth-Z](https://github.com/fabriziofiorucci/NGINX-AuthN-AuthZ) used as example in order to demonstrate how njs [NGINX JavaScript](https://www.nginx.com/products/nginx/modules/nginx-javascript/) can work on top of NGINX Ingress Controller to allow more flexibility and open NIC to serve more specialized use cases. In my example the solution is considered to work on Openshift 4.x but being applied with manifests works also on Kubernetes clusters.

The solution requires a valid NGINX Plus Ingress Controller license and is not intended to work using the OSS version.
To pull the NGINX Plus Ingress Controller image please consider the following article: https://docs.nginx.com/nginx-ingress-controller/installation/pulling-ingress-controller-image/

The solution is meant for deploying NIC on a specific namespace/project, where also the backend-db will run. 


