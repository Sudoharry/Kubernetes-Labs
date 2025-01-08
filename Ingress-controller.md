# Why Use ALB Controller or Ingress Controllers?
---
![image](https://github.com/user-attachments/assets/e16eb8e2-06f3-4886-9def-ce6fff12c059)

---
**ALB Controller** and **Ingress Controllers** are used in Kubernetes for managing external access to the services running inside the cluster. These tools provide a way to expose services using HTTP or HTTPS, balancing the load across multiple backend pods, and managing traffic routing efficiently.

---

## Key Concepts

### ALB Controller (AWS Load Balancer Controller):
- Designed specifically for AWS, it integrates with Elastic Load Balancers (ALBs or NLBs).
- Automatically provisions and manages ALBs in response to Kubernetes Ingress or Service resources.
- Provides features like dynamic scaling, SSL termination, and advanced routing.

### Ingress Controller:
- A general concept in Kubernetes to manage inbound HTTP/HTTPS traffic.
- Allows defining routing rules (based on domain name, URL path, etc.) to distribute traffic to backend services.
- Examples: NGINX Ingress Controller, Traefik, HAProxy.

---

## Real-Time Scenario for Better Understanding
---
![image](https://github.com/user-attachments/assets/70371cc9-fc6b-468e-b290-77fd5ba67cf8)
---
### Scenario: Hosting Multiple Applications on a Single Domain
Imagine you are hosting multiple microservices in a Kubernetes cluster for an e-commerce platform:

- **Service 1**: `catalog-service` to manage product catalogs.
- **Service 2**: `order-service` to process customer orders.
- **Service 3**: `user-service` to handle user authentication.

Each service runs in its own pod, and you want to expose them under a single domain, e.g., `example.com`, with specific routing:

- `example.com/catalog` → `catalog-service`
- `example.com/order` → `order-service`
- `example.com/user` → `user-service`

### Without an Ingress or ALB Controller:
- You would need to expose each service using a separate NodePort or LoadBalancer service, which can be inefficient, costly, and complex to manage.
- Managing SSL certificates for each service would also be a headache.

### With an ALB Controller:
1. **Setup Kubernetes Ingress Resource**:
   - Define routing rules in a Kubernetes Ingress resource for the desired paths.
   - Attach SSL certificates for `example.com`.
2. **ALB Controller Automates Load Balancer Management**:
   - Automatically provisions an AWS ALB.
   - Configures listeners and target groups for the routes and services.
3. **Traffic Routing**:
   - Incoming requests to `example.com/catalog` are routed to `catalog-service` pods.
   - The ALB handles SSL termination, path-based routing, and load balancing between the pods.

### With a General Ingress Controller (e.g., NGINX):
1. Deploy the NGINX Ingress Controller in your Kubernetes cluster.
2. Create an Ingress resource with rules for `example.com` paths.
3. NGINX Ingress Controller will route traffic internally to the respective services based on the paths defined.

---

## Comparison Between ALB Controller and Generic Ingress Controller

| **Feature**            | **ALB Controller**         | **Generic Ingress Controller** |
|-------------------------|----------------------------|---------------------------------|
| **Platform Specificity**| AWS-specific               | Cloud-agnostic                 |
| **Load Balancing**      | Integrates with AWS ALBs   | Uses software-based load balancing (e.g., NGINX) |
| **Ease of Use**         | Simplifies AWS integration | Requires manual setup for cloud-specific features |
| **SSL Termination**     | Managed by AWS             | Requires configuring `cert-manager` or similar tools |
| **Performance**         | High scalability via AWS services | Relies on Kubernetes resources and node capacity |

---

## Conclusion
- Use **ALB Controller** if you're on AWS and want deep integration with AWS services for scalability, cost optimization, and performance.
- Use **NGINX** or other generic ingress controllers if you need a cloud-agnostic solution or prefer flexibility in custom configurations.

---

## Sample Ingress Configuration

### Kubernetes Services
Define services for the applications (`catalog-service`, `order-service`, `user-service`).

```yaml
# catalog-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: catalog-service
  namespace: default
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: catalog
---
# order-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: default
spec:
  ports:
    - port: 80
      targetPort: 8081
  selector:
    app: order
---
# user-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: default
spec:
  ports:
    - port: 80
      targetPort: 8082
  selector:
    app: user
```

### Ingress Resource

#### For AWS ALB Controller:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: alb-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/certificate-arn: <YOUR_ACM_CERTIFICATE_ARN>
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /catalog
            pathType: Prefix
            backend:
              service:
                name: catalog-service
                port:
                  number: 80
          - path: /order
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
          - path: /user
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 80
```

#### For NGINX Ingress Controller:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - example.com
      secretName: example-tls
  rules:
    - host: example.com
      http:
        paths:
          - path: /catalog
            pathType: Prefix
            backend:
              service:
                name: catalog-service
                port:
                  number: 80
          - path: /order
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
          - path: /user
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 80
```

---

### Deploying the Resources

Apply the resources using `kubectl`:

```bash
kubectl apply -f catalog-service.yaml
kubectl apply -f order-service.yaml
kubectl apply -f user-service.yaml
kubectl apply -f <ingress-file-name>.yaml
```

---

### Verifying the Setup

1. Ensure that the Ingress resource is created:
   ```bash
   kubectl get ingress
   ```
2. Check the DNS name (for ALB) or the external IP (for NGINX) and map it to `example.com`.
3. Access the services:
   - `https://example.com/catalog`
   - `https://example.com/order`
   - `https://example.com/user`
