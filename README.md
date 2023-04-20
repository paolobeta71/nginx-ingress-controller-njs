# nginx-ingress-controller-njs example

## Description

This repository is based on [Fabrizio Fiorucci's repository for NGINX-AuthN-Auth-Z](https://github.com/fabriziofiorucci/NGINX-AuthN-AuthZ) used as example in order to demonstrate how njs [NGINX JavaScript](https://www.nginx.com/products/nginx/modules/nginx-javascript/) can work on top of NGINX Ingress Controller to allow more flexibility and open NIC to serve more specialized use cases. In my example the solution is considered to work on Openshift 4.x but being applied with manifests works also on Kubernetes clusters.

The solution requires a valid NGINX Plus Ingress Controller license and is not intended to work using the OSS version.
To pull the NGINX Plus Ingress Controller image please consider the following article: https://docs.nginx.com/nginx-ingress-controller/installation/pulling-ingress-controller-image/  .
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
Change the namespace name with yours. In my example I'm using a ns called project01
```
oc apply -f ns-and-sa.yaml
```
### Deploy the SCC to apply right privileges to the ServiceAccount
```
oc adm policy add-scc-to-user privileged  -z nginx-ingress -n project01
```
### Deploy cluster role and cluster role binding
row 159 is referencing the name of the ns (change with the name of your ns)
```
oc apply -f rbac.yaml
```
### Deploy Optional Default Server Secret
```
oc apply -f default-server-secret.yaml -n project01
```
### Deploying NGINX Config Map
```
oc apply -f nginx-config.yaml -n project01
```
### Deploy the specific Ingress Class
--> be aware, the Ingress Class name is referenced in the 0.backend-db.yaml and 3.nginx-plus-ingress.yaml files
```
oc apply -f ingress-class.yaml -n project01
```
###  Deploy the backend-db App, Service and Ingress
--> change "YOUR_PRIVATE_REGISTRY" with the address of your private registry
```
oc apply -f 0.backend-db.yaml -n project01
```
### Deploy the configmap to apply the specific nginx config
--> a nginx configuration file is mounted using configmap
```
oc apply -f 1.configmap-conf.yaml -n project01
```
### Deploy the configmap to apply the njs script
--> the njs script is mounted using this configmap
```
oc apply -f 2.configmap-js.yaml -n project01
```
### Deploy the NGINX ingress pod
--> check, the two files api.conf and njs.js are mounted using the relative configmaps in the /etc/nginx/conf.d path.

--> change "YOUR_PRIVATE_REGISTRY" with the address of your private registry 
```
oc apply -f 3.nginx-plus-ingress.yaml -n project01
```
### Check all is up and running
```
kubectl get all -n project01
```

## Test the deployment 
[credits always to Fabrizio's repository](https://github.com/fabriziofiorucci/NGINX-AuthN-AuthZ)
Please see also as a reference (https://www.nginx.com/blog/authenticating-api-clients-jwt-nginx-plus/).
This repository's backend DB uses a JWT secret defined as:
```
$ cd jwt
$ cat jwks.json
{
  "keys": [
    {
      "k":"ZmFudGFzdGljand0",
      "kty":"oct",
      "kid":"0001"
    }
  ]
}
```
the k field is the generated symmetric key (base64url-encoded) based on a secret (fantasticjwt in the example). The secret can be generated with the following command:
```
$ echo -n "fantasticjwt" | base64 | tr '+/' '-_' | tr -d '='
ZmFudGFzdGljand0
```
Cloning the repository you have access to the /jwt dir. Create the JWT token using:
```
$ ./jwtEncoder.sh > jwt.token
$ cat jwt.token
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiIsImtpZCI6IjAwMDEiLCJpc3MiOiJCYXNoIEpXVCBHZW5lcmF0b3IiLCJpYXQiOjE2MjYyNTkxOTAsImV4cCI6MTYyNjI1OTE5MX0.eyJuYW1lIjoiSldUIG5hbWUgY2xhaW0iLCJzdWIiOiJKV1Qgc3ViIGNsYWltIiwiaXNzIjoiSldUIGlzcyBjbGFpbSIsInJvbGVzIjpbImd1ZXN0Il19.NbEhykETd6c2wHjU3HDOhypoOCpIGFxC1juZBWKUyO8
```
The decoded token is:
```
{
  "header": {
    "typ": "JWT",
    "alg": "HS256",
    "kid": "0001",
    "iss": "Bash JWT Generator",
    "iat": 1626259190,
    "exp": 1626259191
  },
  "payload": {
    "name": "JWT name claim",
    "sub": "JWT sub claim",
    "iss": "JWT iss claim",
    "roles": [
      "guest"
    ]
  }
}
```
### Backend DB Test
Test the Backend DB fetching the JWT secret:
```
$ curl -s http://db.nginx-authn-authz.ff.lan/jwks.json | jq
{
  "keys": [
    {
      "k": "ZmFudGFzdGljand0",
      "kid": "0001",
      "kty": "oct"
    }
  ]
}
```
Backend DB, fetching all keys:
```
$ curl -s http://db.nginx-authn-authz.ff.lan/backend/fetchallkeys | jq
{
  "rules": [
    {
      "enabled": "true",
      "matchRules": {
        "method": "GET",
        "roles": "guest",
        "xauthz": "api-v1.0"
      },
      "operation": {
        "url": "http://numbersapi.com/random/year"
      },
      "ruleid": 1,
      "uri": "v1.0/getRandomFact"
    },
    {
      "enabled": "true",
      "matchRules": {
        "method": "GET",
        "roles": "guest netops",
        "xauthz": "api-v1.0"
      },
      "operation": {
        "url": "https://api.ipify.org/?format=json"
      },
      "ruleid": 2,
      "uri": "v1.0/getLocalIP"
    },
    {
      "enabled": "true",
      "matchRules": {
        "method": "POST",
        "roles": "devops",
        "xauthz": "api-v2.0"
      },
      "operation": {
        "url": "https://jsonplaceholder.typicode.com/posts"
      },
      "ruleid": 3,
      "uri": "v2.0/testPost"
    }
  ]
}
```
Backend DB, fetching a specific key:
```
$ curl -s http://db.nginx-authn-authz.ff.lan/backend/fetchkey/v1.0/getRandomFact | jq
{
  "rule": {
    "enabled": "true",
    "matchRules": {
      "method": "GET",
      "roles": "guest",
      "xauthz": "api-v1.0"
    },
    "operation": {
      "url": "http://numbersapi.com/random/year"
    },
    "ruleid": 1,
    "uri": "v1.0/getRandomFact"
  }
}
```
### REST API Access test
Display running pods in your specific ns:
```
$ kubectl get pods -n project01
NAME                           READY   STATUS    RESTARTS   AGE
backend-db-5449cd986d-s6wjc    1/1     Running   0          2m35s
nginx-apigw-5b567bd46d-4dzlw   1/1     Running   0          30s
```
Display NGINX Plus IC logs:
```
kubectl logs -l app=nginx-apigw -n project01 -f
```
Open another terminal and use:
```
$ cd jwt
```
Test with valid HTTP method, no JWT token and no X-AuthZ header:
```
$ curl -X GET -ki https://nginx-authn-authz.ff.lan/v1.0/getRandomFact
HTTP/1.1 401 Unauthorized
Server: nginx/1.19.5
Date: Wed, 14 Jul 2021 00:05:58 GMT
Content-Type: text/html
Content-Length: 180
Connection: keep-alive
WWW-Authenticate: Bearer realm="authentication required"

<html>
<head><title>401 Authorization Required</title></head>
<body>
<center><h1>401 Authorization Required</h1></center>
<hr><center>nginx/1.19.10</center>
</body>
</html>
```
Test with valid JWT token, HTTP method and X-AuthZ header:
```
$ curl -X GET -ki -H "X-AuthZ: api-v1.0" -H "Authorization: Bearer `cat jwt.token`" https://nginx-authn-authz.ff.lan/v1.0/getRandomFact
HTTP/1.1 200 OK
Server: nginx/1.19.5
Date: Wed, 14 Jul 2021 10:46:42 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 134
Connection: keep-alive
X-Powered-By: Express
Access-Control-Allow-Origin: *
X-Numbers-API-Number: 1596
X-Numbers-API-Type: year
Pragma: no-cache
Cache-Control: no-cache,no-store, no-cache, must-revalidate, proxy-revalidate, max-age=0
Expires: 0
ETag: W/"86-LOZwbw2FmGZayZAhgGE9bscPWIk"
Last-Modified: 1626259725

1596 is the year that Sir John Norreys and Sir Geoffrey Fenton travel to Connaught to parley with the local Irish lords on June NaNth.
```
Test with valid JWT token, HTTP method and invalid X-AuthZ header:
```
$ curl -X GET -ki -H "X-AuthZ: invalid" -H "Authorization: Bearer `cat jwt.token`" https://nginx-authn-authz.ff.lan/v1.0/getRandomFact
HTTP/1.1 403 Forbidden
Server: nginx/1.19.5
Date: Wed, 14 Jul 2021 10:48:26 GMT
Content-Type: text/html
Content-Length: 154
Connection: keep-alive

<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.19.10</center>
</body>
</html>
```
Test with valid JWT token, X-AuthZ header and invalid HTTP method:
```
$ curl -X POST -ki -H "X-AuthZ: api-v1.0" -H "Authorization: Bearer `cat jwt.token`" https://nginx-authn-authz.ff.lan/v1.0/getRandomFact
HTTP/1.1 403 Forbidden
Server: nginx/1.19.5
Date: Wed, 14 Jul 2021 10:48:46 GMT
Content-Type: text/html
Content-Length: 154
Connection: keep-alive

<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.19.10</center>
</body>
</html>
```
