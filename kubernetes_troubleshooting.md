h1. Kubernetes Troubleshooting: Common Pod-Level Issues

This guide provides step-by-step instructions for troubleshooting common issues that occur with Pods and other Kubernetes workloads. The primary tool used is `kubectl`.

---

h2. Getting Started with Troubleshooting: `kubectl get` and `kubectl describe`

When you encounter an issue with a Kubernetes resource (like a Pod, Deployment, or Service), the first step is almost always to use `kubectl get` and `kubectl describe` to gather initial information.

*   **`kubectl get <resource-type> [resource-name] -n <namespace>`**
    *   **Purpose:** Provides a quick overview of the current state of one or more resources.
    *   **Usage:**
        *   `kubectl get pods`: Lists all pods in the current namespace.
        *   `kubectl get deployments -n my-app-ns`: Lists deployments in a specific namespace.
        *   `kubectl get pod my-app-pod-abcde`: Gets details for a specific pod.
    *   **What to look for:** Pay attention to the `STATUS`, `READY`, `RESTARTS`, and `AGE` columns. These often give immediate clues about the problem.

*   **`kubectl describe <resource-type> <resource-name> -n <namespace>`**
    *   **Purpose:** Provides a detailed view of a specific resource, including its configuration, status, and crucially, a list of recent *Events*.
    *   **Usage:**
        *   `kubectl describe pod my-app-pod-abcde`: Describes a specific pod.
        *   `kubectl describe deployment my-app -n my-app-ns`: Describes a specific deployment.
    *   **What to look for:**
        *   **Events Section:** This is often the most valuable part. Kubernetes records events (like image pull failures, scheduling issues, probe failures) that directly explain why a resource is not behaving as expected.
        *   **Status Conditions:** Look for conditions like `Ready`, `Available`, `Progressing`, and their associated messages.
        *   **Resource Configuration:** Review the `spec` section to ensure the resource is configured as you expect (e.g., image name, ports, volumes, environment variables).

Once you have used `kubectl get` to identify a problematic resource and `kubectl describe` to gather detailed events and status, you can then use the symptoms and diagnoses in the scenarios below to pinpoint the exact issue and find a solution.

---

h2. Scenario 1: Pod is in `ImagePullBackOff` or `ErrImagePull`

*Symptoms:*
A Pod remains in `ImagePullBackOff` or `ErrImagePull` status. This indicates that Kubernetes cannot pull the container image from the specified registry.

{code:bash}
$ kubectl get pods
NAME                             READY   STATUS             RESTARTS   AGE
my-app-pod-5f9f5b8c4-abcde       0/1     ImagePullBackOff   0          2m
{code}

*Troubleshooting Steps:*

1.  **Check Pod Events:** The events section of the Pod will usually provide a clear reason for the image pull failure.
    {code:bash}
    kubectl describe pod my-app-pod-5f9f5b8c4-abcde
    {code}
    Look for events like `Failed to pull image "my-private-registry/my-app:latest": rpc error: code = Unknown desc = Error response from daemon: pull access denied for my-private-registry/my-app, repository does not exist or may require 'docker login'` or `Error: ImagePullBackOff`.

*Common Causes and Solutions:*

*   **Cause A: Image Name or Tag Typo:**
    *   **Diagnosis:** The event message will often say "repository does not exist" or "manifest unknown".
    *   **Solution:** Verify the image name and tag in your Pod or Deployment YAML. Ensure it's spelled correctly and the tag exists in the registry.
*   **Cause B: Private Registry Authentication Failure:**
    *   **Diagnosis:** The event message will indicate "pull access denied" or "unauthorized: authentication required".
    *   **Solution:**
        1.  Ensure you have created a `ImagePullSecret` (e.g., `kubectl create secret docker-registry my-registry-secret --docker-server=<your-registry> --docker-username=<username> --docker-password=<password> --docker-email=<email>`).
        2.  Ensure your Pod's `spec` references this `imagePullSecrets`:
            {code:yaml}
            spec:
              containers:
              - name: my-app
                image: my-private-registry/my-app:latest
              imagePullSecrets:
              - name: my-registry-secret
            {code}
        3.  Verify the secret exists and is correct (`kubectl get secret my-registry-secret -o yaml`).
*   **Cause C: Registry Unreachable:**
    *   **Diagnosis:** The event message might show network-related errors like "connection refused" or "timeout".
    *   **Solution:**
        1.  Check network connectivity from your Kubernetes nodes to the image registry.
        2.  Verify the registry URL is correct.
        3.  Check if there are any firewall rules blocking access.
*   **Cause D: Image Does Not Exist:**
    *   **Diagnosis:** The event message will clearly state "manifest unknown" or "image not found".
    *   **Solution:** Confirm that the image and tag actually exist in the specified registry. You can try to pull it manually from a node using `docker pull <image-name>:<tag>` or `crictl pull <image-name>:<tag>`.

---

h2. Scenario 2: Pod is in `CrashLoopBackOff`

*Symptoms:*
A Pod repeatedly starts and crashes, indicated by the `CrashLoopBackOff` status and an increasing number of restarts.

{code:bash}
$ kubectl get pods
NAME                             READY   STATUS             RESTARTS   AGE
my-app-pod-5f9f5b8c4-abcde       0/1     CrashLoopBackOff   5          5m
{code}

*Troubleshooting Steps:*

1.  **Check Pod Logs:** The most common reason for `CrashLoopBackOff` is an application error. The logs will usually tell you why the application is crashing.
    {code:bash}
    kubectl logs my-app-pod-5f9f5b8c4-abcde
    {code}
    If the pod restarts too quickly, you might need to get logs from a previous instance:
    {code:bash}
    kubectl logs my-app-pod-5f9f5b8c4-abcde --previous
    {code}

2.  **Check Pod Events:** Events can provide information about issues outside the application code, such as failed volume mounts or OOMKills.
    {code:bash}
    kubectl describe pod my-app-pod-5f9f5b8c4-abcde
    {code}

*Common Causes and Solutions:*

*   **Cause A: Application Error/Misconfiguration:**
    *   **Diagnosis:** Logs show application-level errors (e.g., `java.lang.NullPointerException`, `Error: listen EADDRINUSE`).
    *   **Solution:**
        1.  Review your application code for bugs.
        2.  Check application configuration (e.g., environment variables, config maps, secrets) for incorrect values or missing dependencies.
        3.  Ensure the application is designed to run in a containerized environment (e.g., it doesn't try to write to ephemeral storage that gets reset on restart).
*   **Cause B: Missing Dependencies (ConfigMaps, Secrets, Volumes):**
    *   **Diagnosis:** Logs might show errors like "file not found" or "permission denied" when trying to access configuration or data.
    *   **Solution:**
        1.  Verify that all required ConfigMaps, Secrets, and PersistentVolumes are correctly mounted and accessible within the Pod.
        2.  Check permissions on mounted volumes.
*   **Cause C: Liveness Probe Failure:**
    *   **Diagnosis:** Pod events show `Liveness probe failed: HTTP probe failed with statuscode: 500` or similar.
    *   **Solution:**
        1.  Adjust the liveness probe configuration (e.g., `initialDelaySeconds`, `periodSeconds`, `timeoutSeconds`, `failureThreshold`).
        2.  Ensure your application's health endpoint is correctly implemented and returns a successful status (2xx) when healthy.
*   **Cause D: Out Of Memory (OOM) Kill:**
    *   **Diagnosis:** Pod events show `OOMKilled` or `Reason: OOMKilled`.
    *   **Solution:** Refer to **Scenario 3: Pod is OOMKilled** for detailed troubleshooting steps.

---

h2. Scenario 3: Pod is `OOMKilled`

*Symptoms:*
A Pod is terminated by the operating system due to exceeding its memory limits, indicated by the `OOMKilled` status.

{code:bash}
$ kubectl get pods
NAME                             READY   STATUS      RESTARTS   AGE
my-app-pod-5f9f5b8c4-abcde       0/1     OOMKilled   1          10m
{code}

*Troubleshooting Steps:*

1.  **Check Pod Events:** The `describe` command will show the `OOMKilled` event.
    {code:bash}
    kubectl describe pod my-app-pod-5f9f5b8c4-abcde
    {code}
    Look for `Reason: OOMKilled` in the events.

2.  **Check Container Status:** The `kubectl describe pod` output will also show the container's last state, including `OOMKilled: true`.

3.  **Check Logs (if available):** Sometimes, the application might log memory-related issues before being killed.
    {code:bash}
    kubectl logs my-app-pod-5f9f5b8c4-abcde --previous
    {code}

*Common Causes and Solutions:*

*   **Cause A: Insufficient Memory Limits:**
    *   **Diagnosis:** The application genuinely needs more memory than allocated.
    *   **Solution:** Increase the `resources.limits.memory` for the container in your Pod or Deployment YAML. Start with a small increment and monitor.
        {code:yaml}
        resources:
          limits:
            memory: "512Mi" # Increase this value
          requests:
            memory: "256Mi"
        {code}
*   **Cause B: Memory Leak in Application:**
    *   **Diagnosis:** The application's memory usage continuously grows over time, eventually hitting the limit.
    *   **Solution:**
        1.  Monitor memory usage over time using tools like `kubectl top pod` or Prometheus/Grafana if available.
        2.  Analyze application code for memory leaks. Use profiling tools specific to your application's language/framework.
*   **Cause C: Incorrect Memory Calculation:**
    *   **Diagnosis:** The memory usage is higher than expected due to factors like JVM overhead, shared libraries, or sidecar containers.
    *   **Solution:** Account for all memory consumers within the Pod. If using Java, consider `JAVA_TOOL_OPTIONS` to limit heap size.
*   **Cause D: Node Memory Pressure:**
    *   **Diagnosis:** The node itself is running low on memory, leading to aggressive OOM killing of pods.
    *   **Solution:**
        1.  Check node memory usage (`kubectl top node`).
        2.  Consider adding more memory to the node or reducing the number of pods on the node.
        3.  Ensure `requests` are set appropriately to allow the Kubernetes scheduler to place pods on nodes with sufficient available memory.

---

h2. Scenario 4: Pod is Stuck in `Pending`

*Symptoms:*
A Pod remains in the `Pending` state and does not get scheduled to a node.

{code:bash}
$ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
my-app-pod-5f9f5b8c4-abcde       0/1     Pending   0          5m
{code}

*Troubleshooting Steps:*

1.  **Check Pod Events:** The `describe` command is crucial here, as it will show why the scheduler cannot place the Pod.
    {code:bash}
    kubectl describe pod my-app-pod-5f9f5b8c4-abcde
    {code}
    Look for `Events` section for messages like `FailedScheduling`.

*Common Causes and Solutions:*

*   **Cause A: Insufficient Resources (CPU/Memory):**
    *   **Diagnosis:** Events will show messages like `0/3 nodes are available: 3 Insufficient cpu`, `Insufficient memory`.
    *   **Solution:**
        1.  **Increase Cluster Resources:** Add more nodes to your Kubernetes cluster.
        2.  **Adjust Pod Requests:** Reduce the `resources.requests.cpu` or `resources.requests.memory` for the Pod if your application can run with less. Be cautious not to set requests too low, as it can lead to performance issues or `OOMKilled`.
            {code:yaml}
            resources:
              requests:
                cpu: "200m" # Reduce this value
                memory: "256Mi" # Reduce this value
            {code}
        3.  **Check Node Allocatable:** Ensure that the node's allocatable resources are sufficient. Sometimes, system daemons consume a significant portion.
*   **Cause B: Node Selector/Affinity/Taints and Tolerations Mismatch:**
    *   **Diagnosis:** Events will show messages like `0/3 nodes are available: 3 node(s) didn't match node selector`, `node(s) had taints that the pod didn't tolerate`.
    *   **Solution:**
        1.  **Node Selector:** If you are using `nodeSelector`, ensure that there are nodes in the cluster with the matching labels.
            {code:yaml}
            nodeSelector:
              disktype: ssd
            {code}
            Verify node labels: `kubectl get nodes --show-labels`.
        2.  **Taints and Tolerations:** If nodes have taints, ensure your Pod has corresponding tolerations.
            {code:yaml}
            tolerations:
            - key: "key"
              operator: "Exists"
              effect: "NoSchedule"
            {code}
            Check node taints: `kubectl describe node <node-name>`.
        3.  **Node Affinity/Anti-affinity:** Review your `nodeAffinity` or `podAntiAffinity` rules to ensure they are not preventing the Pod from being scheduled.
*   **Cause C: Volume Binding Issues:**
    *   **Diagnosis:** Events will show messages like `waiting for first consumer to be created before binding` or `no persistent volumes available for this claim and no storage class is configured`.
    *   **Solution:**
        1.  **PersistentVolumeClaim (PVC) Pending:** If your Pod is waiting for a PVC, troubleshoot the PVC separately. Refer to **Scenario 9: PersistentVolumeClaim (PVC) Stuck in Pending**.
        2.  **StorageClass:** Ensure a `StorageClass` is defined and correctly configured if you are using dynamic provisioning.
*   **Cause D: Pod Disruption Budget (PDB):**
    *   **Diagnosis:** Less common, but a PDB might prevent a Pod from being scheduled if it would violate the minimum available replicas.
    *   **Solution:** Check PDBs in your namespace: `kubectl get pdb`.
*   **Cause E: Kube-scheduler Issues:**
    *   **Diagnosis:** Very rare, but if the `kube-scheduler` component itself is unhealthy or misconfigured, pods won't be scheduled.
    *   **Solution:** Check the logs of the `kube-scheduler` Pod (usually in `kube-system` namespace).

---

h2. Scenario 5: Pod Fails with `CreateContainerConfigError`

*Symptoms:*
A Pod fails to start with the `CreateContainerConfigError` status.

{code:bash}
$ kubectl get pods
NAME                             READY   STATUS                       RESTARTS   AGE
my-app-pod-5f9f5b8c4-abcde       0/1     CreateContainerConfigError   0          1m
{code}

*Troubleshooting Steps:*

1.  **Check Pod Events:** The `describe` command will provide specific details about why the container configuration failed.
    {code:bash}
    kubectl describe pod my-app-pod-5f9f5b8c4-abcde
    {code}
    Look for `Failed` events related to container creation.

*Common Causes and Solutions:*

*   **Cause A: Missing ConfigMap or Secret:**
    *   **Diagnosis:** Events will show messages like `configmap "my-config" not found` or `secret "my-secret" not found` when trying to mount them as volumes or use them as environment variables.
    *   **Solution:**
        1.  Verify that the referenced ConfigMap or Secret exists in the same namespace as the Pod.
            {code:bash}
            kubectl get configmap my-config -n <namespace>
            kubectl get secret my-secret -n <namespace>
            {code}
        2.  Ensure the name in the Pod's YAML matches the actual ConfigMap/Secret name.
*   **Cause B: Invalid Key in ConfigMap or Secret:**
    *   **Diagnosis:** Events might show `subPath "my-key" not found` or similar, indicating a specific key within the ConfigMap/Secret is missing or misspelled.
    *   **Solution:**
        1.  Inspect the ConfigMap or Secret content to ensure the key exists.
            {code:bash}
            kubectl get configmap my-config -o yaml
            kubectl get secret my-secret -o yaml # Be careful with sensitive data
            {code}
        2.  Correct the key name in the Pod's YAML.
*   **Cause C: SubPath Volume Mount Issues:**
    *   **Diagnosis:** If using `subPath` for volume mounts, the error might indicate that the specified path within the volume doesn't exist.
    *   **Solution:** Ensure the `subPath` refers to a valid directory or file within the mounted volume.
*   **Cause D: Read-Only Root Filesystem:**
    *   **Diagnosis:** If your container image is configured with a read-only root filesystem and the application tries to write to it, you might see this error.
    *   **Solution:**
        1.  Configure your container to write to a mounted volume instead of the root filesystem.
        2.  If necessary, set `readOnlyRootFilesystem: false` in your container spec (though this is generally not recommended for security).
*   **Cause E: SecurityContext Constraints (SCC/PSP - OpenShift/Deprecated):**
    *   **Diagnosis:** In environments like OpenShift, or if using deprecated Pod Security Policies, security contexts might prevent certain operations (e.g., running as root, using host paths).
    *   **Solution:** Review the security context of your Pod and the applicable SCCs/PSPs. Adjust the Pod's security context or request appropriate permissions.

---

h2. Scenario 6: Pod is Unhealthy (Failing Liveness/Readiness Probes)

*Symptoms:*
A Pod is running but is not receiving traffic or is being restarted due to failed health checks (liveness and readiness probes).

{code:bash}
$ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
my-app-pod-5f9f5b8c4-abcde       1/1     Running   0          5m # Readiness probe failing, but pod is running
my-other-pod-abcdefg-hijkl       0/1     Running   1          2m # Liveness probe failing, causing restarts
{code}

*Troubleshooting Steps:*

1.  **Check Pod Events:** The `describe` command will show events related to probe failures.
    {code:bash}
    kubectl describe pod my-app-pod-5f9f5b8c4-abcde
    {code}
    Look for `Liveness probe failed: ...` or `Readiness probe failed: ...` events.

2.  **Check Pod Logs:** The application logs might provide clues if the health endpoint itself is failing.
    {code:bash}
    kubectl logs my-app-pod-5f9f5b8c4-abcde
    {code}

3.  **Test the Probe Endpoint Manually:** If the probe is an HTTP GET, try to access the endpoint from within the cluster (e.g., by `kubectl exec` into another pod and using `curl`).

*Common Causes and Solutions:*

*   **Cause A: Incorrect Probe Configuration:**
    *   **Diagnosis:** The probe path, port, or type (HTTP, TCP, Exec) is incorrect.
    *   **Solution:**
        1.  Verify the `livenessProbe` and `readinessProbe` configuration in your Pod or Deployment YAML.
        2.  Ensure the `path` for HTTP probes is correct and the `port` is the one your application is listening on.
        3.  For `exec` probes, ensure the command exists and returns a 0 exit code for success.
*   **Cause B: Application Not Ready/Healthy:**
    *   **Diagnosis:** The application itself is not responding to health checks, even if the probe configuration is correct.
    *   **Solution:**
        1.  **Readiness Probe:** If the readiness probe is failing, it means your application is not yet ready to serve traffic. This could be due to slow startup, database connections, or other initialization tasks. Consider increasing `initialDelaySeconds` or `periodSeconds` for the readiness probe, or ensure your application truly signals readiness only when it's fully operational.
        2.  **Liveness Probe:** If the liveness probe is failing, your application is unhealthy and needs to be restarted. This often points to an application bug, a deadlock, or resource exhaustion. Review application logs for errors.
*   **Cause C: Resource Exhaustion:**
    *   **Diagnosis:** The application might be running out of CPU or memory, causing it to become unresponsive to health checks.
    *   **Solution:**
        1.  Check `kubectl top pod` for resource usage.
        2.  Increase `resources.limits` for CPU and memory if necessary. Refer to **Scenario 3: Pod is OOMKilled** for memory issues.
*   **Cause D: Network Issues within the Pod:**
    *   **Diagnosis:** Less common, but internal network issues within the Pod (e.g., `localhost` not resolving) could prevent probes from working.
    *   **Solution:** Verify network configuration inside the container. Ensure the application is listening on `0.0.0.0` or the correct interface.

---

h2. Scenario 7: Deployment is Stuck or Not Progressing

*Symptoms:*
A Deployment does not reach its desired number of replicas, or a new rollout gets stuck and doesn't complete.

{code:bash}
$ kubectl get deployments
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
my-app      1/3     1            1           10m # Desired 3, but only 1 ready
{code}

*Troubleshooting Steps:*

1.  **Check Deployment Status:** Use `kubectl get deployment` to see the `READY`, `UP-TO-DATE`, and `AVAILABLE` counts. If `READY` or `AVAILABLE` is less than desired, or `UP-TO-DATE` is not matching desired, there's an issue.

2.  **Check Deployment Events:** The `describe` command for the Deployment will often show rollout-related events.
    {code:bash}
    kubectl describe deployment my-app
    {code}
    Look for `ReplicaSet` creation/deletion events and any warnings.

3.  **Check ReplicaSet Status:** Deployments manage `ReplicaSets`. Check the status of the new `ReplicaSet` (usually with a new hash in its name).
    {code:bash}
    kubectl get replicasets
    {code}

4.  **Check Pod Status:** The most common reason for a Deployment not progressing is that the underlying Pods are failing to start or become ready. Identify the Pods associated with the new ReplicaSet and troubleshoot them.
    {code:bash}
    kubectl get pods -l app=my-app # Use labels to filter
    {code}

*Common Causes and Solutions:*

*   **Cause A: Underlying Pod Issues:**
    *   **Diagnosis:** Pods created by the Deployment are in `Pending`, `ImagePullBackOff`, `CrashLoopBackOff`, `OOMKilled`, or are not becoming `Ready`.
    *   **Solution:** This is the most frequent cause. Troubleshoot the individual Pods using the relevant scenarios in this guide:
        *   **Scenario 1: Pod is in `ImagePullBackOff` or `ErrImagePull`**
        *   **Scenario 2: Pod is in `CrashLoopBackOff`**
        *   **Scenario 3: Pod is `OOMKilled`**
        *   **Scenario 4: Pod is Stuck in `Pending`**
        *   **Scenario 6: Pod is Unhealthy (Failing Liveness/Readiness Probes)**
*   **Cause B: Insufficient Resources for New Pods:**
    *   **Diagnosis:** Similar to Pods stuck in `Pending`, but specifically for new Pods in a rollout. The cluster might not have enough resources to schedule the new replicas.
    *   **Solution:** Refer to **Scenario 4: Pod is Stuck in `Pending`** for solutions related to resource constraints.
*   **Cause C: Readiness Probe Preventing Rollout:**
    *   **Diagnosis:** If the `readinessProbe` is too strict or takes too long, new Pods might never become ready, preventing the Deployment from marking them as available and thus stalling the rollout.
    *   **Solution:** Review and adjust the `readinessProbe` configuration. Ensure it accurately reflects when your application is ready to serve traffic. Refer to **Scenario 6: Pod is Unhealthy (Failing Liveness/Readiness Probes)**.
*   **Cause D: `minReadySeconds` Too High:**
    *   **Diagnosis:** The `minReadySeconds` field in the Deployment spec specifies how long a newly created Pod must be ready before it's considered available. If set too high, it can slow down rollouts.
    *   **Solution:** Review `minReadySeconds` and adjust if necessary. Default is 0.
*   **Cause E: `progressDeadlineSeconds` Exceeded:**
    *   **Diagnosis:** The `progressDeadlineSeconds` defines how long the Deployment controller waits for a Deployment to progress before reporting it as failed. If exceeded, the Deployment will be marked as failed.
    *   **Solution:** Increase `progressDeadlineSeconds` if your application has a very long startup time, or if the underlying issue is a slow rollout due to other factors. Default is 600 seconds.
*   **Cause F: Network Policy Blocking Probes:**
    *   **Diagnosis:** If Network Policies are in place, they might inadvertently block the kubelet from reaching the Pod's health endpoints.
    *   **Solution:** Review your Network Policies to ensure they allow traffic from the kubelet to the Pod's health check ports.

---

h2. Scenario 8: Service Connectivity Issues

*Symptoms:*
Applications within the cluster or external clients cannot connect to a service.

{code:bash}
$ kubectl get svc
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
my-service   ClusterIP   10.96.0.100   <none>        80/TCP     5m
{code}

*Troubleshooting Steps:*

1.  **Verify Service Exists and is Correct:**
    {code:bash}
    kubectl get service my-service -o yaml
    {code}
    Check `selector`, `ports`, and `targetPort`.

2.  **Check Endpoints:** Services rely on Endpoints to route traffic to Pods. If there are no Endpoints, the Service won't work.
    {code:bash}
    kubectl get endpoints my-service
    {code}
    If the output is empty or shows no addresses, the Service is not selecting any Pods.

3.  **Check Pods Selected by Service:** Ensure that Pods matching the Service's `selector` are running and healthy.
    {code:bash}
    kubectl get pods -l <service-selector-key>=<service-selector-value>
    {code}
    (e.g., `kubectl get pods -l app=my-app` if your service selector is `app: my-app`)

4.  **Check Pod Readiness:** Even if Pods are running, they must be `Ready` (i.e., their readiness probes must pass) to be included in the Service's Endpoints.
    {code:bash}
    kubectl get pods -l <service-selector-key>=<service-selector-value>
    {code}
    Look for `READY` column to be `1/1` or similar.

5.  **Check Network Policies:** If Network Policies are enabled in your cluster, they might be blocking traffic to or from the Service.
    {code:bash}
    kubectl get networkpolicies -n <namespace>
    {code}

6.  **Test Connectivity from within the Cluster:**
    *   **From another Pod:** `kubectl exec -it <another-pod> -- curl my-service.<namespace>.svc.cluster.local` (for ClusterIP)
    *   **From a Node:** `curl <service-cluster-ip>:<service-port>`

*Common Causes and Solutions:*

*   **Cause A: No Pods Matching Service Selector:**
    *   **Diagnosis:** `kubectl get endpoints my-service` shows no addresses. The Service's `selector` does not match any running Pods.
    *   **Solution:**
        1.  Verify the `selector` in your Service YAML matches the `labels` on your Pods.
        2.  Ensure the Pods are in the same namespace as the Service.
*   **Cause B: Pods Not Ready:**
    *   **Diagnosis:** Pods are running but their readiness probes are failing, so they are not added to the Service's Endpoints.
    *   **Solution:** Troubleshoot the Pods' readiness probes. Refer to **Scenario 6: Pod is Unhealthy (Failing Liveness/Readiness Probes)**.
*   **Cause C: Incorrect `targetPort`:**
    *   **Diagnosis:** The `targetPort` in the Service definition does not match the port your application is listening on inside the container.
    *   **Solution:** Ensure `targetPort` in the Service YAML matches the `containerPort` defined in your Pod's container spec.
        {code:yaml}
        # Service YAML
        ports:
        - port: 80
          targetPort: 8080 # This must match the container's listening port

        # Pod YAML
        containers:
        - name: my-app
          image: my-image
          ports:
          - containerPort: 8080 # Application listens on this port
        {code}
*   **Cause D: Network Policy Blocking Traffic:**
    *   **Diagnosis:** Connectivity tests fail, and `kubectl describe networkpolicy` shows rules that might be blocking the traffic.
    *   **Solution:** Adjust Network Policies to allow traffic to the Service's Pods from the necessary sources.
*   **Cause E: DNS Resolution Issues:**
    *   **Diagnosis:** `curl` commands using service names fail with "Host not found" or similar.
    *   **Solution:**
        1.  Check `kube-dns` or `CoreDNS` Pods in the `kube-system` namespace.
        2.  Ensure DNS resolution is working within the cluster.
*   **Cause F: External Access (NodePort/LoadBalancer/Ingress) Issues:**
    *   **Diagnosis:** If using `NodePort`, `LoadBalancer`, or `Ingress`, external access fails.
    *   **Solution:**
        1.  **NodePort:** Ensure firewall rules on nodes allow traffic to the NodePort. Check `kubectl get svc -o wide` for the assigned NodePort.
        2.  **LoadBalancer:** Check the cloud provider's load balancer status. Ensure it's provisioned and healthy.
        3.  **Ingress:** Check Ingress controller logs and Ingress resource status. Ensure the Ingress controller is running and correctly configured to route traffic to your Service.

---

h2. Scenario 9: PersistentVolumeClaim (PVC) Stuck in `Pending`

*Symptoms:*
A PersistentVolumeClaim (PVC) remains in the `Pending` state and does not bind to a PersistentVolume (PV).

{code:bash}
$ kubectl get pvc
NAME        STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-pvc      Pending                                      standard       2m
{code}

*Troubleshooting Steps:*

1.  **Check PVC Events:** The `describe` command for the PVC will often provide the reason for its pending state.
    {code:bash}
    kubectl describe pvc my-pvc
    {code}
    Look for `Events` section for messages like `waiting for a volume to be created, either by a storage provisioner or manually created by an administrator`.

2.  **Check StorageClass:** If your PVC specifies a `storageClassName`, ensure that StorageClass exists and is correctly configured.
    {code:bash}
    kubectl get storageclass
    kubectl describe storageclass standard # Replace 'standard' with your StorageClass name
    {code}

3.  **Check Dynamic Provisioner Logs:** If using dynamic provisioning, check the logs of the storage provisioner (e.g., `nfs-subdir-external-provisioner`, `aws-ebs-csi-driver`, `azure-disk-csi-driver`). These are usually Pods in the `kube-system` namespace or a dedicated storage namespace.
    {code:bash}
    kubectl get pods -n kube-system -l app=nfs-subdir-external-provisioner # Example
    kubectl logs <provisioner-pod-name> -n kube-system
    {code}

4.  **Check Existing PersistentVolumes (PVs):** If you are using static provisioning (pre-created PVs), ensure there is an available PV that matches the PVC's requirements.
    {code:bash}
    kubectl get pv
    kubectl describe pv <pv-name>
    {code}

*Common Causes and Solutions:*

*   **Cause A: No Matching PersistentVolume (Static Provisioning):**
    *   **Diagnosis:** If you are not using dynamic provisioning, there might be no pre-created PV that satisfies the PVC's `accessModes`, `storageClassName`, and `capacity` requirements.
    *   **Solution:** Create a PersistentVolume that matches the PVC's criteria. Ensure the `storageClassName` (or lack thereof), `capacity`, and `accessModes` align.
*   **Cause B: StorageClass Not Found or Misconfigured:**
    *   **Diagnosis:** The PVC specifies a `storageClassName` that does not exist, or the StorageClass exists but its provisioner is not running or is misconfigured.
    *   **Solution:**
        1.  Verify the `storageClassName` in the PVC matches an existing StorageClass.
        2.  Ensure the provisioner specified in the StorageClass is installed and running correctly in your cluster.
        3.  Check the StorageClass definition for typos or incorrect parameters.
*   **Cause C: Dynamic Provisioner Issues:**
    *   **Diagnosis:** The storage provisioner Pod is crashing, not running, or its logs show errors when trying to create a volume.
    *   **Solution:**
        1.  Check the status and logs of the storage provisioner Pod. Troubleshoot it like any other Pod (e.g., `CrashLoopBackOff`, `ImagePullBackOff`).
        2.  Ensure the provisioner has the necessary permissions (RBAC) to interact with the underlying storage system (e.g., AWS, Azure, GCP APIs).
*   **Cause D: Insufficient Storage Capacity:**
    *   **Diagnosis:** The underlying storage system (e.g., cloud provider, NFS server) has run out of capacity, and the provisioner cannot create a new volume.
    *   **Solution:** Increase the available storage capacity in your backend storage system.
*   **Cause E: Access Modes Mismatch:**
    *   **Diagnosis:** The PVC requests an `accessMode` (e.g., `ReadWriteMany`) that the available PVs or the StorageClass's provisioner cannot provide.
    *   **Solution:** Adjust the PVC's `accessModes` to match what the storage system supports, or configure your storage system to provide the required access mode.

---

h2. Scenario 10: HelmRelease Reconciliation Failures (Flux CD)

This scenario is specific to users of the Flux CD GitOps tool. A `HelmRelease` is a custom resource that declares a desired state for a Helm chart. When it fails, it means the Helm Operator (part of Flux) could not successfully install or upgrade the chart.

*Symptoms:*

`kubectl get helmrelease` shows that the `READY` status is `False`.

{code:bash}
$ kubectl get helmrelease -n my-app-ns
NAME      AGE   READY   STATUS
my-app    30m   False   reconciliation failed
{code}

*Troubleshooting Steps:*

1.  **Describe the HelmRelease to see the error message.**
    This is the most important command. The status condition at the bottom of the output will contain a detailed error message from the Helm Operator.

    {code:bash}
    kubectl describe helmrelease my-app -n my-app-ns
    {code}

*Cause A: Helm Chart Not Found or Invalid Version*

*Example `describe` output:*
{code}
Status:
  Conditions:
    ...
    Message:  chart "my-chart" version "1.2.4" not found in http://my-chart-repo/index.yaml repository
    Reason:   ChartPullFailed
    Status:   False
    Type:     Ready
{code}

*Diagnosis:*

*   The `Reason: ChartPullFailed` and the message are explicit: the specified chart version does not exist in the Helm repository.

*Solution:*

1.  **Check the `HelmRelease` spec:** Verify the `chart.spec.chart` and `chart.spec.version` fields in your `HelmRelease` YAML.
2.  **Check the `HelmRepository` source:** Ensure the `sourceRef` in your `HelmRelease` points to the correct `HelmRepository` object and that the repository is up-to-date and reachable.
3.  **Check the chart repository:** Manually inspect the Helm repository to see which versions of the chart are available.

*Cause B: Invalid `values` (Helm Template Error)*

*Example `describe` output:*
{code}
Status:
  Conditions:
    ...
    Message:  failed to render chart templates: template: my-chart/templates/deployment.yaml:10:12: executing "my-chart/templates/deployment.yaml" at <.Values.replicaCoun>: nil pointer evaluating interface {}
    Reason:   InstallFailed
    Status:   False
    Type:     Ready
{code}

*Diagnosis:*

*   The error message indicates a Helm templating failure (`failed to render chart templates`).
*   It points to a specific file and line (`deployment.yaml:10:12`) and the source of the error: a typo in the `values` (`replicaCoun` instead of `replicaCount`).

*Solution:*

1.  **Examine the error:** The path to the error is in the message.
2.  **Correct the `values` section** of your `HelmRelease` YAML file. In this case, you would fix the typo from `replicaCoun` to `replicaCount`.
3.  Apply the corrected `HelmRelease`.

*Cause C: Underlying Resources Fail to Become Ready*

*Diagnosis:*

*   The `HelmRelease` `describe` output shows a message like `post-install/post-upgrade hooks failed` or `1/2 deployed resources not ready`.
*   This means Helm successfully installed the chart, but the resources created by the chart (like a Deployment) are failing.

*Solution:*

1.  **The `HelmRelease` is not the root problem.** The issue lies with the Kubernetes objects it created.
2.  **Find the failing resources:** Use `kubectl get all -n my-app-ns` to see the status of all resources created by the release.
3.  **Identify the failing component:** Look for pods in `CrashLoopBackOff`, `ImagePullBackOff`, or deployments that are not progressing.
4.  **Troubleshoot the specific component** using the other scenarios in this guide. For example, if the Deployment created by the Helm chart is stuck, refer to **Scenario 7: Deployment is Stuck or Not Progressing**.
