# Red Hat Consulting Technical Test

# NGINX on OpenShift (Hardened, 2 Replicas)

This repo contains a minimal, production-ready example of deploying an unprivileged NGINX app on OpenShift with best practices:

- **2 replicas**  for HA
- **Health checks** (`/healthz` readiness & liveness)
- **Resource requests/limits** for scheduling & stability
- **ClusterIP Service** for internal access
- **Edge-terminated Route** for external access
- **ConfigMap** for `nginx.conf` and a basic `index.html`
- **NetworkPolicies** with default deny + specific allows (routers)
- **Security**: non-root, no privilege escalation, read-only root filesystem, dropped capabilities, RuntimeDefault seccomp

> Tested with OpenShift 4.x. The image `nginxinc/nginx-unprivileged:1.27-alpine` runs as non-root and listens on 8080.
---

## Quickstart

```shell
# 1) Create project
% oc apply -f 00-namespace.yaml
namespace/red-hat-test created
% oc get project red-hat-test
NAME           DISPLAY NAME   STATUS
red-hat-test                  Active
```
# 2) Apply configs (order matters for clean startup)
```shell
% oc apply -f 01-configmap-nginx.yaml
configmap/nginx-config created
% oc get cm nginx-config
NAME           DATA   AGE
nginx-config   2      13s
% oc apply -f 02-deployment.yaml
deployment.apps/nginx created
% oc get deploy
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   0/2     2            0           9s
% oc get pod -w
NAME                     READY   STATUS    RESTARTS   AGE
nginx-5f56db9954-cdcd8   1/1     Running   0          18s
nginx-5f56db9954-fzj6h   1/1     Running   0          18s
% oc apply -f 03-service.yaml
service/nginx created
% oc get svc
NAME    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
nginx   ClusterIP   10.217.5.214   <none>        80/TCP    11s
% oc describe svc nginx
Name:              nginx
Namespace:         red-hat-test
Labels:            app.kubernetes.io/instance=nginx
                   app.kubernetes.io/name=nginx
                   app.kubernetes.io/part-of=nginx-simple
Annotations:       <none>
Selector:          app.kubernetes.io/instance=nginx,app.kubernetes.io/name=nginx
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.217.5.214
IPs:               10.217.5.214
Port:              http  80/TCP
TargetPort:        8080/TCP
Endpoints:         10.217.0.166:8080,10.217.0.167:8080
Session Affinity:  None
Events:            <none>

oc apply -f 04-route.yaml
route.route.openshift.io/nginx created
% oc get route
NAME    HOST/PORT                             PATH   SERVICES   PORT   TERMINATION     WILDCARD
nginx   nginx-red-hat-test.apps-crc.testing          nginx      http   edge/Redirect   None
% oc describe route
Name:			nginx
Namespace:		red-hat-test
Created:		16 seconds ago
Labels:			app.kubernetes.io/instance=nginx
			app.kubernetes.io/name=nginx
			app.kubernetes.io/part-of=nginx-simple
Annotations:		kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"route.openshift.io/v1","kind":"Route","metadata":{"annotations":{},"labels":{"app.kubernetes.io/instance":"nginx","app.kubernetes.io/name":"nginx","app.kubernetes.io/part-of":"nginx-simple"},"name":"nginx","namespace":"red-hat-test"},"spec":{"port":{"targetPort":"http"},"tls":{"insecureEdgeTerminationPolicy":"Redirect","termination":"edge"},"to":{"kind":"Service","name":"nginx","weight":100}}}
			
			openshift.io/host.generated=true
Requested Host:		nginx-red-hat-test.apps-crc.testing
			   exposed on router default (host router-default.apps-crc.testing) 16 seconds ago
Path:			<none>
TLS Termination:	edge
Insecure Policy:	Redirect
Endpoint Port:		http

Service:	nginx
Weight:		100 (100%)
Endpoints:	10.217.0.166:8080, 10.217.0.167:8080
```

# 3) Lock down traffic with NetworkPolicies
```shell
% oc apply -f 05-networkpolicy-deny-all.yaml
networkpolicy.networking.k8s.io/default-deny-all created
% oc get networkpolicy
NAME               POD-SELECTOR   AGE
default-deny-all   <none>         12s
% oc describe networkpolicy default-deny-all
Name:         default-deny-all
Namespace:    red-hat-test
Created on:   2025-08-30 19:02:12 -0400 AST
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     <none> (Allowing the specific traffic to all pods in this namespace)
  Allowing ingress traffic:
    <none> (Selected pods are isolated for ingress connectivity)
  Not affecting egress traffic
  Policy Types: Ingress

% oc apply -f 06-networkpolicy-allow-from-openshift-ingress.yaml
networkpolicy.networking.k8s.io/allow-from-openshift-ingress created
% oc get networkpolicy                                     
NAME                           POD-SELECTOR                                                    AGE
allow-from-openshift-ingress   app.kubernetes.io/instance=nginx,app.kubernetes.io/name=nginx   6s
default-deny-all               <none>                                                          96s
% oc describe networkpolicy  allow-from-openshift-ingress 
Name:         allow-from-openshift-ingress
Namespace:    red-hat-test
Created on:   2025-08-30 19:03:42 -0400 AST
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     app.kubernetes.io/instance=nginx,app.kubernetes.io/name=nginx
  Allowing ingress traffic:
    To Port: 8080/TCP
    From:
      NamespaceSelector: network.openshift.io/policy-group=ingress
  Not affecting egress traffic
  Policy Types: Ingress

```
# 4) Tests from the Openshift Route to validate access.
```shell
oc run curl  --rm -it  --image=ubuntu:latest -- /bin/bash  
root@curl:/# apt update
root@curl:/# apt install curl -y
  root@curl:/# curl -v -k https://nginx-red-hat-test.apps-crc.testing
* Host nginx-red-hat-test.apps-crc.testing:443 was resolved.
* IPv6: (none)
* IPv4: 192.168.127.2
*   Trying 192.168.127.2:443...
* Connected to nginx-red-hat-test.apps-crc.testing (192.168.127.2) port 443
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_128_GCM_SHA256 / X25519 / RSASSA-PSS
* ALPN: server did not agree on a protocol. Uses default.
* Server certificate:
*  subject: CN=*.apps-crc.testing
*  start date: Jul 10 10:46:13 2025 GMT
*  expire date: Jul 10 10:46:14 2027 GMT
*  issuer: CN=ingress-operator@1752144372
*  SSL certificate verify result: self-signed certificate in certificate chain (19), continuing anyway.
*   Certificate level 0: Public key type RSA (2048/112 Bits/secBits), signed using sha256WithRSAEncryption
*   Certificate level 1: Public key type RSA (2048/112 Bits/secBits), signed using sha256WithRSAEncryption
* using HTTP/1.x
> GET / HTTP/1.1
> Host: nginx-red-hat-test.apps-crc.testing
> User-Agent: curl/8.5.0
> Accept: */*
> 
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* old SSL session ID is stale, removing
< HTTP/1.1 200 OK
< server: nginx/1.27.5
< date: Sat, 30 Aug 2025 23:13:13 GMT
< content-type: text/html
< content-length: 662
< last-modified: Sat, 30 Aug 2025 22:56:15 GMT
< etag: "68b3818f-296"
< accept-ranges: bytes
< set-cookie: dc11feb31783fd8e2a34ee298d360621=169461f3f889410c8c26912d8b87b13f; path=/; HttpOnly; Secure; SameSite=None
< cache-control: private
< 
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8"/>
  <title>OpenShift NGINX</title>
  <meta name="viewport" content="width=device-width, initial-scale=1"/>
  <style>
    body{font-family:system-ui,-apple-system,Segoe UI,Roboto,Helvetica,Arial,sans-serif;margin:2rem;line-height:1.4}
    .card{max-width:680px;padding:1.25rem;border:1px solid #e5e7eb;border-radius:12px}
    code{background:#f3f4f6;padding:.2rem .4rem;border-radius:6px}
  </style>
</head>
<body>
  <div class="card">
    <h1>✅ NGINX on OpenShift</h1>
    <p>If you can read this, the deployment works.</p>
    <p>Health check: <code>/healthz</code></p>
  </div>
</body>
</html>
* Connection #0 to host nginx-red-hat-test.apps-crc.testing left intact
```
# Test the nginx service directly from a pod within the same namespace:
# This connection test fails because the default-deny-all network policy is being applied.

```shell
root@curl:/# curl -v -k http://nginx.red-hat-test.svc.cluster.local
* Host nginx.red-hat-test.svc.cluster.local:80 was resolved.
* IPv6: (none)
* IPv4: 10.217.5.214
*   Trying 10.217.5.214:80...
* connect to 10.217.5.214 port 80 from 10.217.0.176 port 53552 failed: Connection timed out
* Failed to connect to nginx.red-hat-test.svc.cluster.local port 80 after 130067 ms: Couldn't connect to server
* Closing connection
curl: (28) Failed to connect to nginx.red-hat-test.svc.cluster.local port 80 after 130067 ms: Couldn't connect to server
```


