# Helm Chart Documentation

## Overview

This Helm chart deploys a multi-component application that includes:

- A front-facing web application (Frontend).
- A non-public web application (Backend).
- A database for data storage (PostgreSQL).

The chart includes high availability, secure secret management, and an IP whitelist configuration for specific routes.

---

## Installation Instructions

### Prerequisites

- Kubernetes cluster
- Kubectl installed and configured with access to the cluster
- Helm installed (v3+)


### Steps

1. Clone the repository or copy the chart to your system.

   ```bash
   git clone https://github.com/mzmor/vindsburg.git
   cd vindsburg
   ```

2. Optional: Update the `values.yaml` file to match your requirements.

3. Install the chart:

   ```bash
   helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
   helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
   helm repo update
   helm dependency update
   helm install vindsburg .
   ```
   Next: Seal the secrets and upgrade the chart - see [sealing instructions](sealing.md)

4. Verify the deployment:

   ```bash
   kubectl get all
   ```

5. Ensure DNS or `/etc/hosts` points to the ingress IP for `frontend.example.com`. In my case, I added Minikube IP to the `/etc/hosts` file:

   ```bash
   # If running on Minikube:
   echo "$(minikube ip) frontend.example.com" >> /etc/hosts

   # Verify DNS:
   nslookup frontend.example.com
   ```

6. Test the application:

   ```bash
   curl -i http://frontend.example.com/
   ```

---

## Key Architectural Choices

### 1. Database

**PostgreSQL** was chosen for its robustness and compatibility with a wide range of applications. The PostgreSQL container image (`postgres:15`) is stable, well-documented, and widely used. Deployed as StatefulSet with 2 replicas to ensure:
- stable storage: Each pod gets its own PersistentVolume. Storage remains attached even if pod is rescheduled. Data persists across pod restarts.
- stable identity: Each pod gets a predictable and persistent name: `database-0`, `database-1`. If DNS is used, DNS entries remain stable across pod restarts.

### 2. Ingress Controller

The chart is designed to work with the **NGINX Ingress Controller** given its popularity and support for advanced features such as:

- IP whitelisting.
- Flexible annotations.
- SSL/TLS termination.

IP whitelisting is implemented via separate files as `ingress-admin.yaml` and `ingress-api-internal.yaml` for better granularity of access control.

`ingress-root.yaml` is the configuration for public access to http://frontend.example.com/.

### 3. Secrets Management

Sensitive data, such as database credentials, is stored as Sealed Secrets. These secrets are referenced in the `deployment-database.yaml` and `backend-deployment.yaml` files using environment variables to ensure secure handling.

*Sealed Secrets* tool was chosen to manage secrets. It encrypts secrets and stores them safely in the git repository. Kubernetes handles the decryption. The encryption mechanism is assymetric. The sealed key is stored in the git repository. While private key is stored in the cluster (only accessible by the Sealed Secrets controller).

### 4. High Availability

High availability is achieved through:

- **Replica Count**: All deployments are configured with a default replica count of 2 (customizable via `values.yaml`).
- **Autoscaling**: An HPA (Horizontal Pod Autoscaler) is included for the Frontend and Backend application, scaling pods based on CPU utilization.
- **Resource Requests and Limits**: Resource requests and limits are set to ensure optimal resource allocation basing on CPU tests performed with `hey` tool. The target for optimal latency was < 100ms for 95% of requests. 100000 requests were sent in 200 concurrent connections. The target performance was achieved with 800m CPU request for frontend and backend. Notice that CPU limit was intentionally not set according to best practices (CPU is a compressible resource, when hitting the limit, pods get throttled. Throttling can happen even if node has available CPU - wasting resources).

```bash
$ hey -n 100000 -c 200 http://frontend.example.com/

Summary:
  Total:        15.0564 secs
  Slowest:      0.3797 secs
  Fastest:      0.0010 secs
  Average:      0.0294 secs
  Requests/sec: 6641.7025
  
  Total data:   51579896 bytes
  Size/request: 515 bytes

Response time histogram:
  0.001 [1]     |
  0.039 [76406] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.077 [19991] |■■■■■■■■■■
  0.115 [2581]  |■
  0.152 [686]   |
  0.190 [208]   |
  0.228 [84]    |
  0.266 [24]    |
  0.304 [12]    |
  0.342 [3]     |
  0.380 [4]     |


Latency distribution:
  10% in 0.0087 secs
  25% in 0.0145 secs
  50% in 0.0240 secs
  75% in 0.0377 secs
  90% in 0.0553 secs
  95% in 0.0696 secs
  99% in 0.1153 secs

Details (average, fastest, slowest):
  DNS+dialup:   0.0000 secs, 0.0010 secs, 0.3797 secs
  DNS-lookup:   0.0000 secs, 0.0000 secs, 0.0165 secs
  req write:    0.0000 secs, 0.0000 secs, 0.0172 secs
  resp wait:    0.0292 secs, 0.0009 secs, 0.3793 secs
  resp read:    0.0002 secs, 0.0000 secs, 0.0176 secs

Status code distribution:
  [200] 100000 responses
```


### 5. IP Whitelisting

- `/admin`: Access restricted to IPs in the range `192.168.0.0/24`.
- `/api/internal`: Restricted to internal network IPs (`10.0.0.0/16`).

These configurations are applied via annotations in `ingress-admin.yaml` and `ingress-api-internal.yaml` files. You can customize the ranges in `values.yaml`.

---

### Verification Steps

#### Post-Deployment Testing

1. **Check Pod Status**: Ensure all pods are running:

   ```bash
   kubectl get pods

   NAME                                                  READY   STATUS    RESTARTS   AGE
   backend-6cf947fd88-k4b97                              1/1     Running   0          19m
   backend-6cf947fd88-mkh5q                              1/1     Running   0          19m
   database-0                                            1/1     Running   0          19m
   database-1                                            1/1     Running   0          15m
   frontend-66489cf7-hwv2g                               1/1     Running   0          19m
   frontend-66489cf7-ml67v                               1/1     Running   0          19m
   sealed-secrets-65d77bb647-9bh5p                       1/1     Running   0          19m
   vindsburg-ingress-nginx-controller-5ccc89f664-d7r4s   1/1     Running   0          19m
   ```

2. **Ingress Access**:

   - Test the `frontend.example.com` endpoint:
      ```bash
      curl -i http://frontend.example.com

      HTTP/1.1 200 OK
      Date: Fri, 24 Jan 2025 08:32:31 GMT
      Content-Type: text/plain; charset=utf-8
      Content-Length: 480
      Connection: keep-alive

      Hostname: backend-ddc475f7f-d6vvj
      IP: 127.0.0.1
      IP: ::1
      IP: 10.244.0.89
      IP: fe80::c051:68ff:fe4f:1bea
      RemoteAddr: 10.244.0.91:40570
      GET / HTTP/1.1
      Host: frontend.example.com
      User-Agent: curl/7.81.0
      Accept: */*
      Connection: close
      X-Forwarded-For: 192.168.49.1, 10.244.0.97
      X-Forwarded-Host: frontend.example.com
      X-Forwarded-Port: 80
      X-Forwarded-Proto: http
      X-Forwarded-Scheme: http
      X-Real-Ip: 10.244.0.97
      X-Request-Id: 804b4f618b50b660d7556d7f04972530
      X-Scheme: http
      ```

   - Verify that `/admin` and `/api/internal` are accessible only from whitelisted IPs.
      ```
      curl -i http://frontend.example.com/admin
      HTTP/1.1 403 Forbidden
      Date: Fri, 24 Jan 2025 10:58:54 GMT
      Content-Type: text/html
      Content-Length: 146
      Connection: keep-alive

      <html>
      <head><title>403 Forbidden</title></head>
      <body>
      <center><h1>403 Forbidden</h1></center>
      <hr><center>nginx</center>
      </body>
      </html>
      ```
   - Set your internal IP to the whitelist range in `values.yaml` file and test:
      ```
      curl -i http://frontend.example.com/api/internal
      HTTP/1.1 200 OK
      Date: Fri, 24 Jan 2025 10:58:45 GMT
      Content-Type: text/plain; charset=utf-8
      Content-Length: 493
      Connection: keep-alive

      Hostname: backend-ddc475f7f-h6qf4
      IP: 127.0.0.1
      IP: ::1
      IP: 10.244.0.142
      IP: fe80::546f:1cff:fe86:913
      RemoteAddr: 10.244.0.144:57872
      GET /api/internal HTTP/1.1
      Host: frontend.example.com
      User-Agent: curl/7.81.0
      Accept: */*
      Connection: close
      X-Forwarded-For: 192.168.49.1, 10.244.0.97
      X-Forwarded-Host: frontend.example.com
      X-Forwarded-Port: 80
      X-Forwarded-Proto: http
      X-Forwarded-Scheme: http
      X-Real-Ip: 10.244.0.97
      X-Request-Id: 209db6a9cf3342dc3a02f80fec312d81
      X-Scheme: http
      ```

3. **Database Connectivity**:

   - Connect to the database pod and verify access:
     ```bash
     kubectl exec -it database-0 -- psql -d app_db -U app_user

     app_db-# \l
                                                List of databases
        Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges   
     -----------+----------+----------+------------+------------+------------+-----------------+-----------------------
      app_db    | app_user | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | 
      postgres  | app_user | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | 
      template0 | app_user | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/app_user          +
                |          |          |            |            |            |                 | app_user=CTc/app_user
      template1 | app_user | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/app_user          +
                |          |          |            |            |            |                 | app_user=CTc/app_user
      (4 rows)
     ```

4. **Autoscaling**: Simulate high CPU usage and ensure that HPA scales pods as expected.

   - Frontend testing:
   
     Example stress test on frontend:

      ```bash
      # Get the frontend pod name
      kubectl get pods
      # Run stress test
      kubectl exec -it <frontend-pod-name> -- /bin/sh
      # Install strees-ng tool on Alpine Linux
      apk update
      apk add stress-ng
      # Run stress test
      stress-ng --cpu 4 --cpu-load 90 --timeout 600s
      ```

      In another terminal, check if the HPA scaled the frontend pods (here, HPA scaled from 2 to 3 pods):
      ```bash
      kubectl get hpa frontend-hpa

      NAME           REFERENCE             TARGETS        MINPODS   MAXPODS   REPLICAS   AGE
      frontend-hpa   Deployment/frontend   cpu: 67%/80%   2         10        3          39m

      kubectl get pods

      NAME                                                  READY   STATUS    RESTARTS      AGE
      backend-ddc475f7f-h6qf4                               1/1     Running   0             67m
      backend-ddc475f7f-kscnf                               1/1     Running   0             67m
      database-0                                            1/1     Running   0             67m
      database-1                                            1/1     Running   0             67m
      frontend-8555bc5c9f-7hqds                             1/1     Running   0             38s
      frontend-8555bc5c9f-f9947                             1/1     Running   0             67m
      frontend-8555bc5c9f-pf6dv                             1/1     Running   0             67m
      vindsburg-ingress-nginx-controller-5ccc89f664-qq6ch   1/1     Running   2 (22h ago)   67m
      ```

      When the stress test is finished, the HPA will scale down the frontend pods to the original number of replicas (2):
      ```bash
      kubectl get hpa frontend-hpa
      NAME           REFERENCE             TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
      frontend-hpa   Deployment/frontend   cpu: 0%/80%   2         10        2          54m
      ```

   - Backend and frontend testing:

     Ensure `hey` is installed (in my case on Ubuntu):
      ```bash
      sudo snap install hey
      ```

      Send 100000 requests in 200 concurrent connections to the backend through the frontend:
      ```bash
      hey -n 100000 -c 200 http://frontend.example.com/
      ```

      Watch as Horizontal Pod Autoscalers scale as well backend and frontend pods:
      ```bash
      kubectl get hpa
      NAME           REFERENCE             TARGETS        MINPODS   MAXPODS   REPLICAS   AGE
      backend-hpa    Deployment/backend    cpu: 57%/80%   2         10        2          18m
      frontend-hpa   Deployment/frontend   cpu: 7%/80%    2         10        2          112m

      NAME           REFERENCE             TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
      backend-hpa    Deployment/backend    cpu: 183%/80%   2         10        5          19m
      frontend-hpa   Deployment/frontend   cpu: 41%/80%    2         10        2          112m

      NAME           REFERENCE             TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
      backend-hpa    Deployment/backend    cpu: 133%/80%   2         10        10         20m
      frontend-hpa   Deployment/frontend   cpu: 146%/80%   2         10        3          113m

      NAME           REFERENCE             TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
      backend-hpa    Deployment/backend    cpu: 0%/80%   2         10        2          27m
      frontend-hpa   Deployment/frontend   cpu: 0%/80%   2         10        2          120m
      ```
---

