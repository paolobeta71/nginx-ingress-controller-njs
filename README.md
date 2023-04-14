# nginx-ingress-controller-njs example

## Description

This repository is based on [Fabrizio Fiorucci's repository for NGINX-AuthN-Auth-Z](https://github.com/fabriziofiorucci/NGINX-AuthN-AuthZ) used as example in order to demonstrate how njs [NGINX JavaScript](https://www.nginx.com/products/nginx/modules/nginx-javascript/) can work on top of NGINX Ingress Controller to allow more flexibility and open NIC to serve more specialized use cases. In my example the solution is considered to work on Openshift 4.x but being applied with manifests works also on Kubernetes clusters.

The solution requires a valid NGINX Plus Ingress Controller license and is not intended to work using the OSS version.
To pull the NGINX Plus Ingress Controller image please consider the following article: https://docs.nginx.com/nginx-ingress-controller/installation/pulling-ingress-controller-image/
The solution is meant for deploying NIC on a specific namespace/project, where also the backend-db will run. 

## Requirements
- An Openshift or Kubernetes cluster.
- The NGINX Plus Ingress Controller Image available from a private registry. It can include also AppProtect.
- The backend-db image available from a private registry (to build it please see the instructions below). 

## Deploying the Repository
### Build the backend-db:
```
cd backend-db
docker build --no-cache -t YOUR_PRIVATE_REGISTRY/nginx-authn-authz-backend-db:1.0 .
docker push YOUR_PRIVATE_REGISTRY/nginx-authn-authz-backend-db:1.0
```
### Deploy the Namespace and Service Account
In my example I'm using a ns called project01
```
oc apply -f ns-and-sa.yaml
```
### Deploy the SCC to apply right privileges to the ServiceAccount
```
oc adm policy add-scc-to-user privileged  -z nginx-ingress -n project01
```
### Deploy cluster role and cluster role binding
```
oc apply -f rbac.yaml
```
 
