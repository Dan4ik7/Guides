# Configuring PostgreSQL Access via NGINX Ingress Controller in Kubernetes

This guide explains how to configure PostgreSQL access through an NGINX ingress controller in Kubernetes. By default, the NGINX ingress controller is designed to handle HTTP and HTTPS traffic (Layer 7) and does not natively support TCP or UDP traffic (Layer 4). To enable TCP traffic (e.g., PostgreSQL on port `5432`), additional configuration is required, such as using a `tcp-services-configmap` and modifying the ingress controller deployment, service, and network policies.

---

## **Preconditions**
Before proceeding, ensure the following prerequisites are met:
1. **Kubernetes Cluster**: You have a running Kubernetes cluster with `kubectl` configured to interact with it.
2. **NGINX Ingress Controller**: The NGINX ingress controller is installed and running in the cluster.
   - Verify the ingress controller deployment:
     ```bash
     kubectl get deployments -n nginx-system
     ```
3. **PostgreSQL Deployment**: PostgreSQL is deployed in the cluster and accessible via a Kubernetes service.
   - Verify the PostgreSQL service:
     ```bash
     kubectl get svc -n postgresql
     ```
4. **TCP Services ConfigMap**: The NGINX ingress controller is configured to use a `tcp-services-configmap` for forwarding TCP traffic.
5. **Network Policies**: Network policies are enabled in the cluster and can be modified as needed.

---

## **Steps**

### **1. Add Arguments to the NGINX Ingress Controller Deployment**
In order for the NGINX ingress controller to handle TCP traffic, it must be explicitly configured to use a `tcp-services-configmap`. This is done by adding the `--tcp-services-configmap` argument to the ingress controller deployment.

#### **Steps:**
1. Edit the NGINX ingress controller deployment:
   ```bash
   kubectl edit deployment nginx-nginx-ingress-controller -n nginx-system
   ```
2. Add the `--tcp-services-configmap` argument to the container's `args` section:
   ```yaml
   spec:
     containers:
     - args:
       - --default-backend-service=$(POD_NAMESPACE)/nginx-nginx-ingress-controller-default-backend
       - --http-port=8080
       - --https-port=8443
       - --tcp-services-configmap=nginx-system/nginx-nginx-ingress-controller
   ```
   - **Why?** This argument tells the ingress controller to look for a ConfigMap that defines how to handle TCP traffic. Without this, the ingress controller will only handle HTTP/HTTPS traffic.

3. Save and exit the editor.

4. Restart the NGINX ingress controller to apply the changes:
   ```bash
   kubectl rollout restart deployment nginx-nginx-ingress-controller -n nginx-system
   ```

---

### **2. Update the ConfigMap for TCP Services**
The NGINX ingress controller uses a ConfigMap to define TCP services. This ConfigMap maps external ports (e.g., `5432`) to internal Kubernetes services (e.g., the PostgreSQL service).

#### **Steps:**
1. Edit the `tcp-services-configmap` for the NGINX ingress controller:
   ```bash
   kubectl edit configmap nginx-nginx-ingress-controller -n nginx-system
   ```
2. Add the following entry under `data`:
   ```yaml
   data:
     "5432": "postgresql/my-release-postgresql:5432"
   ```
   - **Why?** This entry maps port `5432` on the ingress controller to the PostgreSQL service running in the `postgresql` namespace. Without this mapping, the ingress controller will not know how to forward TCP traffic to PostgreSQL.

3. Save and exit the editor.

4. Restart the NGINX ingress controller to apply the changes:
   ```bash
   kubectl rollout restart deployment nginx-nginx-ingress-controller -n nginx-system
   ```

---

### **3. Expose Port on the NGINX Service**
The NGINX ingress controller service must expose the required port (`5432`) so that external clients can connect to it.

#### **Steps:**
1. Edit the NGINX ingress controller service:
   ```bash
   kubectl edit svc nginx-nginx-ingress-controller -n nginx-system
   ```
2. Add the required port (`5432`) to the `spec.ports` section:
   ```yaml
   spec:
     ports:
     - name: http
       port: 80
       protocol: TCP
       targetPort: 80
     - name: https
       port: 443
       protocol: TCP
       targetPort: 443
     - name: postgres
       port: 5432
       protocol: TCP
       targetPort: 5432
   ```
   - **Why?** By default, the NGINX ingress controller service only exposes ports `80` and `443`. Adding port `5432` ensures that PostgreSQL traffic can reach the ingress controller.

3. Save and exit the editor.

4. Verify the updated service:
   ```bash
   kubectl describe svc nginx-nginx-ingress-controller -n nginx-system
   ```
   Ensure that port `5432` is listed under the `Ports` section.

---

### **4. Allow TCP Traffic on the NGINX Ingress Controller NetworkPolicy**
If a `NetworkPolicy` is applied to the NGINX ingress controller, it must allow ingress traffic on port `5432`.

#### **Steps:**
1. Edit the `nginx-nginx-ingress-controller` NetworkPolicy:
   ```bash
   kubectl edit networkpolicy nginx-nginx-ingress-controller -n nginx-system
   ```
2. Add the required port (`5432`) to the `ingress` section:
   ```yaml
   spec:
     ingress:
     - ports:
       - port: 8080
         protocol: TCP
       - port: 8443
         protocol: TCP
       - port: 5432
         protocol: TCP
       from:
       - {}
   ```
   - **Why?** Without this rule, the NGINX ingress controller will block incoming traffic on port `5432`, preventing external clients from connecting to PostgreSQL.

3. Save and exit the editor.

4. Verify the updated NetworkPolicy:
   ```bash
   kubectl describe networkpolicy nginx-nginx-ingress-controller -n nginx-system
   ```
   Ensure that port `5432` is listed under the `Allowing ingress traffic` section.

---

### **5. Update the PostgreSQL NetworkPolicy**
The PostgreSQL `NetworkPolicy` must allow ingress traffic from the NGINX ingress controller.

#### **Steps:**
1. Edit the `my-release-postgresql` NetworkPolicy:
   ```bash
   kubectl edit networkpolicy my-release-postgresql -n postgresql
   ```
2. Add the following rule to allow traffic from the NGINX ingress controller:
   ```yaml
   spec:
     ingress:
     - from:
       - namespaceSelector:
           matchLabels:
             app.kubernetes.io/instance: nginx
       ports:
       - protocol: TCP
         port: 5432
   ```
   - **Why?** This ensures that the PostgreSQL pods accept traffic from the NGINX ingress controller on port `5432`.

3. Save and exit the editor.

4. Verify the updated NetworkPolicy:
   ```bash
   kubectl describe networkpolicy my-release-postgresql -n postgresql
   ```
   Ensure that port `5432` is listed under the `Allowing ingress traffic` section.

---

### **6. Test the Configuration**
After completing the above steps, test the connection to PostgreSQL:

1. From your host machine:
   ```bash
   psql -h <nginx-external-ip> -p 5432 -U postgres -d postgres
   ```
   Replace `<nginx-external-ip>` with the external IP of the NGINX ingress controller.

2. Using `nc`:
   ```bash
   nc -zv <nginx-external-ip> 5432
   ```

---

### **Summary**
- **Step 1:** Add arguments to the NGINX ingress controller deployment to use the `tcp-services-configmap`.
- **Step 2:** Update the `tcp-services-configmap` to map PostgreSQL traffic.
- **Step 3:** Ensure the NGINX ingress controller service exposes port `5432`.
- **Step 4:** Allow TCP traffic on the NGINX ingress controller NetworkPolicy.
- **Step 5:** Update the PostgreSQL NetworkPolicy to allow traffic from the NGINX ingress controller.
- **Step 6:** Test the configuration to confirm connectivity.

---

This guide ensures that PostgreSQL is accessible via the NGINX ingress controller in Kubernetes
