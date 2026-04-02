# Kubernetes Troubleshooting & Error Handling Guide

Kubernetes is complex so failures can happen at many overlapping layers: Infrastructure (Nodes/Network), orchestration (Controllers), execution (Containers), or logic (Application configuration).

The primary goal of this guide is to establish a **Methodical Troubleshooting Mindset**. When something breaks, do not guess. Knowing *how to think* and *where to look* is 90% of the battle.

---

## 1. The Methodical Troubleshooting Mindset

When an application fails, you must isolate the failure layer. Always approach Kubernetes using the **OSI Model / Onion Model**—moving from the outside-in (Top-Down) or inside-out (Bottom-Up).

### Where to Look First Based on the Symptom:
1. **"The browser says 'This site cannot be reached' or 'NXDOMAIN'"** -> Stop looking at Kubernetes. This is a **DNS or Domain Registrar** issue.
2. **"The browser yields 502 Bad Gateway or 404"** -> The DNS works. Look at **AWS Load Balancers (ALB) and Kubernetes Ingress**.
3. **"Traffic gets to the pod, but the app crashes or gives 500 Internals"** -> Ingress works. Look at **Pod Logs and Application Code**.
4. **"Pod refuses to start (CrashLoop or Pending)"** -> Look at **Kubernetes Events and Configuration (Secrets/ConfigMaps/Quotas)**.

### The 4 Essential Investigative Commands
Always run these in this order when a pod is misbehaving:
1. `kubectl get pods -A` (Identify the high-level state: `Running`, `Pending`, `CrashLoopBackOff`)
2. `kubectl describe pod <pod-name>` (Scroll to the **Events** section. This shows what the Kubernetes Control Plane is complaining about: Limits, Volumes, Scheduling).
3. `kubectl logs <pod-name> --previous` (Shows the actual Container/Application crash stack trace).
4. `kubectl get events --sort-by='.metadata.creationTimestamp'` (See a timeline of all cluster operations).

---

## 2. Phase 1: External & Edge Routing (The "Healthy Cluster" Illusion)

Sometimes `kubectl get po -o wide`, `get svc`, and `get ing` all show healthy, green resources, but the website is completely unreachable.

### Error: Reconciled ALB, Healthy Pods, but Browser times out
*   **Context:** You have an ALB Ingress Controller. `kubectl get ing` shows the ALB address, your pods are 1/1 Running, and the Service exists.
*   **Cause (The Hostinger Scenario):** The issue is outside the cluster. If you just bought the domain (`yassinabuelsheikh.store`) or just pointed its A-Record/CNAME to the AWS ALB, DNS propagation across the global internet takes time (from a few minutes to 24 hours). 
*   **Fix & Method:**
    1.  *Rule out the cluster:* Run `nslookup yassinabuelsheikh.store` or `dig yassinabuelsheikh.store` on your terminal.
    2.  If the resolved IP does not match the IP of your AWS ALB (`k8s-sharedalb...elb.amazonaws.com`), it is a global DNS propagation delay. You just have to wait. 

### Error: `502 Bad Gateway` from the ALB
*   **Cause:** The ALB exists and routed the traffic to the node, but the NodePort or Pod port is closed/wrong.
*   **Fix:** Ensure the `targetPort` in your `.yaml` Service matches the exact port exposed by your Dockerfile. 

---

## 3. Phase 2: Configuration Constraints & Resource Limits

If you configured cluster governance (using Terraform or Helm), these safety nets often become blind spots.

### Error: Pods endlessly crashing or `OOMKilled` on startup (The Prometheus Scenario)
*   **Context:** You deploy a notoriously heavy application like Prometheus. It immediately goes into `CrashLoopBackOff` or `OOMKilled`.
*   **Cause:** While provisioning the environment, you used Terraform to apply a restrictive `ResourceQuota` or `LimitRange` to the namespace. The pod requires 2GB of RAM, but the namespace forcibly limits pods to 512MB. 
*   **Methodical Troubleshooting:**
    1.  Run `kubectl describe pod <prometheus-pod>`. Read the "Events". It will explicitly state `exceeded quota: memory-limit`.
    2.  Check the namespace constraints: `kubectl describe resourcequota -n monitoring` or `kubectl get limitrange -n monitoring`.
*   **Fix:** Adjust the Terraform code that generated the Namespace limits to accommodate observation tech stacks (which naturally need higher CPU/Memory).

---

## 4. Phase 3: External Secrets and IAM (IRSA) Bridging

Connecting EKS to AWS Secrets Manager or RDS is the #1 point of failure in cloud-native deployments because it spans multiple abstraction layers.

### Error: `fe_sendauth: no password supplied`
*   **Context:** The backend logs show an RDS connection failure because there is no password. You *know* the database is accessible because the network packet reached RDS successfully. You also *know* the pod has an AWS IAM Service Account (IRSA) attached.
*   **Cause:** Assigning the IAM Role string to the `ServiceAccount` is only step 1. Kubernetes itself does not natively read AWS Secrets. If you just define the IRSA role, the pod has *permission* to see the secret, but no mechanism to *fetch* it into an environment variable.
*   **Methodical Troubleshooting:**
    1.  **Check the Bridging Objects:** Did you deploy the External Secrets Operator definitions?
        ```bash
        kubectl get secretstore,externalsecret -n backend
        ```
    2.  If `kind: SecretStore` (which tells K8s *where* to get the secret, e.g., AWS parameter store) and `kind: ExternalSecret` (which tells K8s *what* specific key to fetch) are missing, the native Kubernetes `Secret` is never generated.
    3.  **Check the final output:** Always run `kubectl describe secret <generated-secret-name>`. If the base64 value is missing, your ExternalSecret mapping failed.

---

## 5. Phase 4: Container Startup & "Ghost Pods"

### Error: Pod `NotFound` right after you see it
*   **Cause:** Kubernetes ReplicaSets are relentless. A pod crashes, and the ReplicaSet instantly kills it and spins up a brand new one with a new random ID string. 
*   **Methodical Troubleshooting:**
    *   Stop copy-pasting Pod IDs that change every 30 seconds.
    *   Use Labels: `kubectl describe po -n frontend -l app=frontend`.
    *   Find out why the previous iteration died: `kubectl logs -n frontend -l app=frontend --previous`.

### Error: Nginx `host not found in upstream`
*   **Context:** Nginx crashes immediately on boot because it cannot resolve `backend-service`.
*   **Cause:** Nginx evaluates DNS on the very first millisecond of startup. If the backend Kubernetes Service doesn't exist yet, or if it is deployed in a different namespace, the local DNS query fails.
*   **Methodical Troubleshooting:**
    *   Did you deploy them in order?
    *   If frontend is in the `frontend` namespace and backend is in the `backend` namespace, `backend-service` will fail. You MUST use the FQDN in your Nginx config:
      `http://backend-service.backend.svc.cluster.local`.

---

## 6. Phase 5: Storage and Persistent Volumes (Common Pitfalls)

### Error: `Multi-Attach error for volume "pvc-xyz"`
*   **Context:** You update a deployment that uses an AWS EBS Volume (PersistentVolumeClaim). The old pod starts terminating, but the new pod gets stuck in `ContainerCreating` forever.
*   **Cause:** AWS EBS Volumes are strictly `ReadWriteOnce`—they can only be attached to **one EKS Node at a time**. During a rolling deployment, Kubernetes tries to spin up the new pod on Node B while the terminating pod is still slowly shutting down on Node A. AWS refuses to let Node B mount the volume until Node A completely releases it.
*   **Fix:**
    *   Wait gracefully (it resolves in ~2 minutes).
    *   For rapid deployments of stateful apps using EBS, change the Deployment strategy from `RollingUpdate` to `Recreate`:
      ```yaml
      strategy:
        type: Recreate
      ```
      This forces K8s to completely kill the old pod (and unmount the volume) *before* it even tries to spin up the new one.

---

## 7. Phase 6: The "What If" Edge Cases (Project-Specific Risks)

Even if you didn't experience these explicitly in this run, your specific architecture (ArgoCD, ECR, ALB, Redis) makes you highly susceptible to these common failures:

### "What If" 1: Pod is Running `0/1 Ready` (Not CrashLoopBackOff)
*   **Cause (Liveness/Readiness Probes):** The container started successfully so it doesn't crash, but the application is taking too long to boot (e.g., Django running migrations or Redis loading snapshot to memory). The `ReadinessProbe` checks `/api/health/` and receives a timeout. Kubernetes refuses to send traffic to the pod until the probe returns HTTP `200`.
*   **Methodical Troubleshooting:** Check `kubectl describe po`. Look at the bottom under Events for `Readiness probe failed: HTTP probe failed with statuscode: 500`. Increase the `initialDelaySeconds` in your deployment YAML so Kubernetes waits longer before checking if the app is ready.

### "What If" 2: `ErrImagePull` or `ImagePullBackOff`
*   **Cause (ECR Permissions):** If you are using AWS ECR for your Docker images, the EKS Node Group (AWS EC2 instances) must have the `AmazonEC2ContainerRegistryReadOnly` IAM policy attached. If you forgot this in your Terraform `eks infra` code, the nodes cannot authenticate to ECR to pull the image.
*   **Methodical Troubleshooting:** Run `kubectl describe po`. If the Events show `pull access denied` or `no basic auth credentials`, it's an IAM issue on the Node Group, NOT a Kubernetes issue.

### "What If" 3: ArgoCD says `OutOfSync` but the cluster functions perfectly
*   **Cause (Configuration Drift):** Someone bypassed GitOps and ran a manual `kubectl edit deployment backend` or `kubectl scale` command directly in the terminal to quickly fix a production issue. ArgoCD detected that the live cluster state no longer matches the source of truth in the GitHub repository (`k8s-manifests`).
*   **Fix:** Use the ArgoCD UI "App Diff" to find what was manually changed. Update the GitHub repository to reflect the needed change, or click "Sync" in ArgoCD to overwrite the manual terminal change and enforce the Git repo's true state.

### "What If" 4: ALB Ingress Controller refuses to create an ALB
*   **Cause (Subnet Tagging):** You deployed your `ingress.yaml` but `kubectl get ing` shows `<none>` under Address for 15+ minutes. The AWS ALB Controller requires your AWS VPC Subnets to be specifically tagged so it knows *where* it is mathematically allowed to place the physical ALB. 
*   **Fix:** Ensure your Terraform VPC module tagged your public subnets with `kubernetes.io/role/elb = 1` and private with `kubernetes.io/role/internal-elb = 1`. Always check the controller logs if Ingress hangs: 
  ```bash
  kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
  ```
