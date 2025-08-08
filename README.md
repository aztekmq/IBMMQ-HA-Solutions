## Objective
The goal is to create a local Kubernetes environment using Minikube to simulate an IBM MQ Native HA deployment, mimicking the architecture described in the [AWS Partner Solution for IBM MQ](https://aws-ia.github.io/cfn-ps-ibm-mq/). This POC will:
- Deploy a Kubernetes cluster locally.
- Install IBM MQ Advanced for Developers with Native HA (one active queue manager, two standby replicas).
- Use persistent storage to simulate Amazon EBS.
- Simulate a load balancer for client connections.
- Demonstrate HA failover, message persistence, and basic queue operations.
- Provide a foundation for transitioning to a production EKS setup.

---

## Prerequisites
Ensure the following tools and resources are available on your laptop:

1. **Minikube** (v1.33 or later):
   - Install: [Minikube installation guide](https://minikube.sigs.k8s.io/docs/start/)
   - Verify: `minikube version`
   - Minimum requirements: 8GB RAM, 4 CPU cores, 20GB free disk space.

2. **kubectl** (v1.28 or later):
   - Install: [kubectl installation guide](https://kubernetes.io/docs/tasks/tools/)
   - Verify: `kubectl version --client`

3. **Docker** (v20.10 or later):
   - Install: [Docker installation guide](https://docs.docker.com/get-docker/)
   - Verify: `docker --version`
   - Ensure Docker is running.

4. **Helm** (v3.12 or later):
   - Install: [Helm installation guide](https://helm.sh/docs/intro/install/)
   - Verify: `helm version`

5. **IBM MQ Client Tools** (optional, for testing):
   - Download the IBM MQ Advanced for Developers client from [IBM’s website](https://www.ibm.com/products/mq/downloads) to test queue operations (e.g., `amqsputc`, `amqsget`).
   - Alternatively, use a programming language (e.g., Python with `pymqi`) for client testing.

6. **System Requirements**:
   - Operating System: Windows, macOS, or Linux.
   - Internet access to pull container images and Helm charts.
   - Administrative privileges to modify `/etc/hosts` (or equivalent) for Ingress testing.

7. **IBM MQ Container Image**:
   - Use the IBM MQ Advanced for Developers image (`ibmcom/mq:9.4.3`) for non-production use.
   - For production, you’d need a licensed IBM MQ image, but the developer image is sufficient for this POC.

---

## Step-by-Step Implementation

### Step 1: Set Up Minikube
Minikube creates a single-node Kubernetes cluster on your laptop, simulating the EKS control plane and worker nodes.

1. **Start Minikube**:
   ```bash
   minikube start --memory=6144 --cpus=4 --driver=docker --kubernetes-version=v1.28.3
   ```
   - `--memory=6144`: Allocates 6GB RAM to handle IBM MQ’s resource needs.
   - `--cpus=4`: Allocates 4 CPU cores for stability.
   - `--driver=docker`: Uses Docker as the virtualization driver (use `virtualbox` or `hyperkit` if preferred).
   - `--kubernetes-version=v1.28.3`: Matches a stable version compatible with IBM MQ Helm charts.
   - *Note*: Adjust resource allocations if your laptop has limited capacity, but ensure at least 4GB RAM and 2 CPUs.

2. **Verify Cluster**:
   ```bash
   minikube status
   kubectl cluster-info
   ```
   Output should confirm the cluster is running, with the API server accessible at `https://<minikube-ip>:8443`.

3. **Enable Minikube Add-ons**:
   Enable the Ingress add-on to simulate a load balancer and the storage-provisioner for persistent storage:
   ```bash
   minikube addons enable ingress
   minikube addons enable storage-provisioner
   ```

4. **Get Minikube IP**:
   Record the Minikube IP for later use:
   ```bash
   minikube ip
   ```
   Example output: `192.168.49.2`. This IP will be used for accessing services.

---

### Step 2: Configure Persistent Storage
IBM MQ Native HA requires persistent storage for queue manager data to ensure message durability across failovers. We’ll use Minikube’s hostpath provisioner to simulate Amazon EBS.

1. **Create a Storage Class**:
   Create a file named `mq-storageclass.yaml`:
   ```yaml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: mq-storage
   provisioner: k8s.io/minikube-hostpath
   reclaimPolicy: Retain
   volumeBindingMode: Immediate
   ```
   Apply it:
   ```bash
   kubectl apply -f mq-storageclass.yaml
   ```
   This defines a storage class for persistent volumes, with `Retain` policy to preserve data after pod deletion.

2. **Create Persistent Volume Claims (PVCs)**:
   IBM MQ Native HA requires shared storage for queue manager data. Create a file named `mq-pvc.yaml`:
   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: mq-data
     namespace: mq
   spec:
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 2Gi
     storageClassName: mq-storage
   ```
   *Note*: The `mq` namespace will be created in Step 3. The `ReadWriteOnce` mode is sufficient for Minikube’s single-node setup, though production EKS deployments may use `ReadWriteMany` for multi-node access.

---

### Step 3: Deploy IBM MQ Native HA
We’ll use the IBM MQ Helm chart to deploy a Native HA queue manager with three replicas (one active, two standby), as described in the AWS Partner Solution.

1. **Create a Namespace**:
   Create a dedicated namespace for IBM MQ:
   ```bash
   kubectl create namespace mq
   ```

2. **Add IBM MQ Helm Repository**:
   Add the IBM MQ Helm chart repository:
   ```bash
   helm repo add ibm-mq https://ibm-messaging.github.io/mq-helm
   helm repo update
   ```
   *Note*: If the repository is unavailable, download the sample Helm chart from the [IBM MQ AWS Partner Solution GitHub](https://github.com/aws-ia/cfn-ps-ibm-mq) or IBM’s official documentation.

3. **Pull IBM MQ Container Image** (optional):
   Verify access to the IBM MQ Advanced for Developers image:
   ```bash
   docker pull icr.io/ibm-messaging/mq:9.4.3
   ```
   This ensures the image is available locally, reducing deployment delays.

4. **Create Helm Values File**:
   Create a file named `mq-values.yaml` to configure the Native HA setup:
   ```yaml
   image:
     repository: icr.io/ibm-messaging/mq
     tag: 9.4.3
     pullPolicy: IfNotPresent
   queueManager:
     name: QUICKSTART
     license: accept
     version: 9.4.3
     replicas: 3
     storage:
       persistence:
         enabled: true
         storageClassName: mq-storage
         size: 2Gi
     highAvailability:
       enabled: true
       topologySpreadConstraints:
         - maxSkew: 1
           topologyKey: kubernetes.io/hostname
           whenUnsatisfiable: ScheduleAnyway
           labelSelector:
             matchLabels:
               app.kubernetes.io/name: mq-quickstart
     metrics:
       enabled: true
   service:
     type: LoadBalancer
     port: 1414
   web:
     enabled: true
     port: 9443
     security:
       enabled: true
       tls:
         enabled: true
   ```
   Key settings:
   - `replicas: 3`: Deploys three queue managers (one active, two standby).
   - `highAvailability.enabled: true`: Enables Native HA, ensuring automatic failover.
   - `storage.persistence`: Uses the `mq-storage` storage class for persistent data.
   - `service.type: LoadBalancer`: Simulates the AWS Classic Load Balancer for client connections.
   - `web.enabled: true`: Enables the IBM MQ web console for monitoring.
   - `topologySpreadConstraints`: Ensures replicas are distributed (though limited in Minikube’s single-node setup).

5. **Apply the PVC**:
   Apply the PVC created earlier, now that the namespace exists:
   ```bash
   kubectl apply -f mq-pvc.yaml
   ```

6. **Deploy IBM MQ**:
   Install the Helm chart in the `mq` namespace:
   ```bash
   helm install mq-quickstart ibm-mq/ibm-mq -f mq-values.yaml --namespace mq
   ```
   This deploys the IBM MQ queue manager with Native HA.

7. **Verify Deployment**:
   Check pod status:
   ```bash
   kubectl get pods -n mq
   ```
   Expected output:
   ```
   NAME                    READY   STATUS    RESTARTS   AGE
   mq-quickstart-0         1/1     Running   0          2m
   mq-quickstart-1         1/1     Running   0          2m
   mq-quickstart-2         1/1     Running   0          2m
   ```
   Check services:
   ```bash
   kubectl get svc -n mq
   ```
   Look for the `mq-quickstart` service with type `LoadBalancer`.

---

### Step 4: Configure Load Balancing and Access
In the AWS Partner Solution, a Classic Load Balancer routes client connections to the active queue manager. Minikube doesn’t provide a real load balancer, so we’ll use the Ingress add-on and port forwarding to simulate this.

1. **Create an Ingress Resource**:
   Create a file named `mq-ingress.yaml`:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: mq-ingress
     namespace: mq
   spec:
     rules:
     - host: mq.local
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: mq-quickstart
               port:
                 number: 1414
     - host: mq-console.local
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: mq-quickstart-web
               port:
                 number: 9443
   ```
   Apply it:
   ```bash
   kubectl apply -f mq-ingress.yaml
   ```

2. **Update Hosts File**:
   Add entries to your laptop’s `/etc/hosts` (Linux/macOS) or `C:\Windows\System32\drivers\etc\hosts` (Windows):
   ```text
   <minikube-ip> mq.local
   <minikube-ip> mq-console.local
   ```
   Replace `<minikube-ip>` with the output of `minikube ip` (e.g., `192.168.49.2`).

3. **Access the IBM MQ Service**:
   - **Queue Manager**: Connect to `mq.local:1414` using an MQ client.
   - **Web Console**: Access the IBM MQ web console at `https://mq-console.local:9443/ibmmq/console`.
     - If the console is inaccessible, use port forwarding:
       ```bash
       kubectl port-forward svc/mq-quickstart-web 9443:9443 -n mq
       ```
       Then open `https://localhost:9443/ibmmq/console` in a browser.
     - Default credentials (if not customized): Username `admin`, Password `passw0rd` (check Helm chart documentation for defaults).

4. **Alternative: LoadBalancer Access**:
   If the Helm chart’s LoadBalancer service works, get its external IP:
   ```bash
   kubectl get svc mq-quickstart -n mq
   ```
   In Minikube, use `minikube service mq-quickstart --url -n mq` to get a URL for the service:
   ```bash
   minikube service mq-quickstart --url -n mq
   ```
   This provides a temporary URL (e.g., `http://192.168.49.2:xxxxx`) for testing.

---

### Step 5: Test IBM MQ Native HA
To validate the POC, test the core HA features: failover, message persistence, and load balancing.

1. **Set Up an IBM MQ Client**:
   Install the IBM MQ client tools or use a sample application. For example, use the `amqsputc` and `amqsget` tools from the IBM MQ toolkit:
   - Download from [IBM MQ Downloads](https://www.ibm.com/products/mq/downloads).
   - Configure the client with the following connection details:
     - Host: `mq.local` or the LoadBalancer URL.
     - Port: `1414`.
     - Queue Manager: `QUICKSTART`.
     - Channel: Default is `DEV.APP.SVRCONN` (check Helm chart for specifics).

2. **Send and Receive Test Messages**:
   - Create a test queue using the web console or `runmqsc`:
     ```bash
     kubectl exec -it mq-quickstart-0 -n mq -- runmqsc QUICKSTART
     DEFINE QLOCAL(TEST.QUEUE)
     END
     ```
   - Send messages using `amqsputc`:
     ```bash
     amqsputc TEST.QUEUE QUICKSTART
     ```
     Enter a few test messages (e.g., `Hello MQ`), then press Ctrl+D.
   - Retrieve messages using `amqsget`:
     ```bash
     amqsget TEST.QUEUE QUICKSTART
     ```

3. **Test Failover**:
   Simulate a failure by deleting the active queue manager pod:
   ```bash
   kubectl delete pod mq-quickstart-0 -n mq
   ```
   - Kubernetes will restart the pod, and one of the standby queue managers (`mq-quickstart-1` or `mq-quickstart-2`) will become active.
   - Verify the new active pod:
     ```bash
     kubectl get pods -n mq
     ```
   - Check the web console or use `runmqsc` to confirm the queue manager status:
     ```bash
     kubectl exec -it <new-active-pod> -n mq -- runmqsc QUICKSTART
     DISPLAY QMSTATUS
     ```
   - Send/receive messages again to confirm message persistence.

4. **Verify Load Balancing**:
   - Repeatedly connect to `mq.local:1414` using the MQ client.
   - The Ingress or LoadBalancer service should route connections to the active queue manager.
   - Monitor pod logs to confirm which pod handles the connection:
     ```bash
     kubectl logs mq-quickstart-0 -n mq
     ```

5. **Monitor via Web Console**:
   - Access the web console (`https://mq-console.local:9443/ibmmq/console` or `https://localhost:9443/ibmmq/console` via port forwarding).
   - Check queue manager status, queues, and message counts.
   - Verify that the active queue manager is serving requests and standby queue managers are ready.

---

### Step 6: Validate Core Concepts
This POC demonstrates the following IBM MQ Native HA concepts, aligned with the AWS EKS deployment:

1. **High Availability**:
   - Three queue manager replicas ensure one is active, with two in standby.
   - Automatic failover occurs when the active pod is deleted, with minimal disruption.
   - Shared persistent storage ensures message durability.

2. **Persistent Storage**:
   - The `mq-storage` storage class and PVC simulate Amazon EBS, preserving queue data across pod restarts.
   - Messages sent to `TEST.QUEUE` remain available after failover.

3. **Load Balancing**:
   - The Ingress or LoadBalancer service routes client connections to the active queue manager, mimicking the AWS Classic Load Balancer.
   - Clients experience seamless connectivity during failover.

4. **Kubernetes Orchestration**:
   - Minikube simulates EKS’s managed Kubernetes environment, handling pod scheduling, scaling, and restarts.
   - Helm charts simplify IBM MQ deployment and configuration.

5. **Containerization**:
   - The IBM MQ container image runs queue managers in a Kubernetes-native way, consistent with EKS deployments.

---

### Step 7: Clean Up
When the POC is complete, remove the resources to free up your laptop’s resources:

1. **Uninstall IBM MQ**:
   ```bash
   helm uninstall mq-quickstart -n mq
   ```

2. **Delete Namespace and Resources**:
   ```bash
   kubectl delete namespace mq
   kubectl delete -f mq-storageclass.yaml
   ```

3. **Stop and Delete Minikube**:
   ```bash
   minikube stop
   minikube delete
   ```

4. **Remove Hosts File Entries**:
   Remove the `mq.local` and `mq-console.local` entries from `/etc/hosts`.

---

### Additional Considerations
1. **Transition to EKS**:
   To deploy this in AWS EKS, replicate the setup with these changes:
   - Create an EKS cluster using `eksctl` or the AWS Management Console.
   - Use Amazon EBS CSI driver for persistent storage.
   - Deploy a Classic Load Balancer or Application Load Balancer (ALB) instead of Ingress.
   - Configure IAM roles for EKS and MQ, as per the [AWS Partner Solution](https://aws-ia.github.io/cfn-ps-ibm-mq/).
   - Use a licensed IBM MQ container image for production.

2. **Security**:
   - For the POC, TLS is enabled for the web console but not for client connections. In production, configure TLS for all channels using Kubernetes secrets.
   - Set custom admin credentials in the Helm values file:
     ```yaml
     web:
       security:
         adminUser: your-username
         adminPassword: your-password
     ```

3. **Performance**:
   - Minikube’s single-node setup limits scalability. Monitor resource usage with:
     ```bash
     kubectl top pods -n mq
     ```
   - If pods fail due to resource constraints, increase Minikube’s memory/CPU allocation or reduce `replicas` to 2.

4. **IBM MQ Operator**:
   - For advanced management, consider the IBM MQ Operator (preview available for EKS: [IBM MQ Operator EKS Preview](https://github.com/ibm-messaging/mq-operator-eks-preview-2025)).
   - The operator simplifies queue manager configuration but requires additional setup.

5. **Monitoring**:
   - Enable metrics in the Helm chart (already included in `mq-values.yaml`) and integrate with Prometheus/Grafana for monitoring in a production setup.
   - For the POC, use the web console for basic monitoring.

---

### Troubleshooting
- **Pod Status Issues**:
   - If pods are in `CrashLoopBackOff`, check logs:
     ```bash
     kubectl logs mq-quickstart-0 -n mq
     ```
     Common issues: Invalid license, insufficient storage, or resource limits.
   - Ensure the `license: accept` setting is in `mq-values.yaml`.

- **Service Unreachable**:
   - Verify the service and Ingress:
     ```bash
     kubectl get svc -n mq
     kubectl get ingress -n mq
     ```
   - Check `/etc/hosts` for correct Minikube IP mapping.

- **Storage Errors**:
   - Confirm the PVC is bound:
     ```bash
     kubectl get pvc -n mq
     ```
   - If unbound, ensure the storage class exists and Minikube’s storage-provisioner is enabled.

- **Web Console Access**:
   - If `https://mq-console.local:9443` fails, use port forwarding and check TLS certificate issues in the browser.
   - Verify credentials in the Helm chart documentation.

---

### Sample Test Workflow
To demonstrate the POC’s functionality:
1. Access the web console (`https://localhost:9443/ibmmq/console` via port forwarding).
2. Create a queue (`TEST.QUEUE`) using the console or `runmqsc`.
3. Send 10 test messages using `amqsputc TEST.QUEUE QUICKSTART`.
4. Delete the active pod (`kubectl delete pod mq-quickstart-0 -n mq`).
5. Wait for failover (30-60 seconds) and verify the new active pod.
6. Retrieve messages using `amqsget TEST.QUEUE QUICKSTART` to confirm persistence.
7. Monitor the process in the web console.

---

### Conclusion
This detailed guide sets up a local Minikube cluster to simulate an IBM MQ Native HA environment, replicating key aspects of an AWS EKS deployment. The POC demonstrates high availability, persistent storage, load balancing, and Kubernetes orchestration, providing a solid foundation for understanding IBM MQ Native HA. You can extend this setup to EKS by adapting the Helm charts and configurations for AWS resources.
