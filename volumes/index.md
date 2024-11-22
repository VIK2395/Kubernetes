- Network-backed storage (nfs storage)
- Cloud storage

Kubernetes doesn't provide data persistance out-of-the-box

- 1\. **We need a storrage that doesn't depend on the pod lifecycle**
- 2\. **Storage must be available across all nodes**
- 3\. **Storage must survive cluster crash**

## Local vs Remote Volumes

Local violates 2 and 3 requirements

**Local storage** in Kubernetes refers to storage that is **tied to a specific node** in the cluster, rather than being abstracted or distributed across multiple nodes (as with persistent volumes from cloud providers or networked storage solutions). It is typically used for scenarios where data locality is important or performance is critical, such as caching or temporary storage for a single pod.

### **Key Features of Local Storage**
1. **Node Affinity**: Pods using local storage must run on the node where the local storage is physically located. Kubernetes ensures this by scheduling the pod on the appropriate node.
2. **Performance**: Since it uses local disks, it can provide faster read/write speeds and lower latency compared to remote storage.
3. **Non-Portable**: The data cannot be easily moved between nodes. If the node fails, the data may become unavailable unless a backup mechanism is in place.

---

### **Types of Local Storage in Kubernetes**

Kubernetes filesystem definitions
- **Node's Local Filesystem**: Think of this as a **sandboxed workspace** Kubernetes sets up for pods. It’s temporary, isolated, and managed by Kubernetes for specific tasks.
- **Node's Host Filesystem**: This is the **entire computer’s hard drive**, where Kubernetes has no direct control unless explicitly configured (e.g., with HostPath).

Storages
1. **EmptyDir**:
   - **Tied to the pod** on a specific node and exists on the **node's local storage(node's local filesystem)** (e.g., `/var/lib/kubelet`).
     - The **node's local filesystem** refers to the physical or virtual disk storage **on the host machine where the Kubernetes node is running**.
     - An `EmptyDir` volume creates temporary storage for a pod on the **node's local filesystem**, usually under `/var/lib/kubelet/pods/<pod-id>/volumes/`
     - Even though the data resides on the **host machine**, Kubernetes abstracts it away, so pods interact with it as if it were their own **local storage**.
     - It's managed by Kubernetes.
     - The data exists on the node, but Kubernetes manages it and isolates it for pod use.
   - A temporary storage volume that is created for a pod.
   - It exists only as long as the pod is running.
   - `EmptyDir` provides a shared and persistent directory for **all containers within a pod**.
   - The directory is created on the **node's local disk** (or memory if configured with medium: Memory).
   - Even if one container in the pod restarts, the data in the EmptyDir volume remains intact and accessible by other containers in the same pod.
   - **Persistence Within the Pod**: Even if a container restarts, the EmptyDir data persists as long as the pod exists.
   - **Shared Storage for Containers**: All containers within the same pod can read/write data to the EmptyDir volume, enabling communication and data sharing.
   - The volume is dedicated to the pod that created it, meaning **other pods cannot access it**, even if they are running on the same node.

   Example:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: emptydir-example
   spec:
     containers:
     - name: app
       image: nginx
       volumeMounts:
       - name: data-volume
         mountPath: /data
     volumes:
     - name: data-volume
       emptyDir: {}
   ```

3. **HostPath**:
   - Tied to a specific directory or file on the **host's filesystem**.
   - A **HostPath** volume gives the pod direct access to a specific file or directory on the **node's host filesystem**.
   - A volume that mounts a file or directory from **the host node's** filesystem into the pod.
     - **Security Concerns**: Exposes the **host's filesystem**, which could lead to potential risks if improperly configured.
   - **Lifecycle**: The data is tied to the host node, not the pod. Even if the pod is deleted, the data persists on the host.
   - **Location**: Directly uses the host node's filesystem, making it non-ephemeral and potentially shared across pods on the same node.
   - The data is **independent of the pod’s lifecycle** and persists on the host **even after the pod is deleted**.
   - If the pod is rescheduled to a different node, the data on the original node does not move with the pod, and a new instance of the pod won't have access to that data unless it’s rescheduled on the same node.
   - Multiple pods running on the same node can access the same HostPath directory if they are configured to use it.
   - This allows the pod to read/write data directly on the host machine.

   Example:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: hostpath-example
   spec:
     containers:
     - name: app
       image: nginx
       volumeMounts:
       - name: host-volume
         mountPath: /data
     volumes:
     - name: host-volume
       hostPath:
         path: /data-on-host
         type: DirectoryOrCreate
   ```

5. **Local Persistent Volumes**:
   - A more advanced and persistent form of local storage.
   - Unlike `emptyDir` or `hostPath`, it integrates with Kubernetes' Persistent Volume (PV) and Persistent Volume Claim (PVC) system.
   - It provides higher durability and is suitable for stateful workloads like databases.

   Example:
   ```yaml
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: local-pv
   spec:
     capacity:
       storage: 10Gi
     accessModes:
     - ReadWriteOnce
     persistentVolumeReclaimPolicy: Retain
     storageClassName: local-storage
     local:
       path: /mnt/disks/ssd1
     nodeAffinity:
       required:
         nodeSelectorTerms:
         - matchExpressions:
           - key: kubernetes.io/hostname
             operator: In
             values:
             - node1
   ```

   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: local-pvc
   spec:
     accessModes:
     - ReadWriteOnce
     resources:
       requests:
         storage: 10Gi
     storageClassName: local-storage
   ```

---

### **Use Cases for Local Storage**
1. **High-Performance Workloads**: Applications needing low latency and high IOPS.
2. **Ephemeral Data**: Data that does not need to persist beyond the lifecycle of a pod.
3. **Caching or Temporary Data**: Data that can be reconstructed or regenerated.

---

### **Limitations of Local Storage**
1. **Non-Resilient**: Local storage is tied to a node, so if the node fails, the data may be lost.
2. **Scheduling Constraints**: Kubernetes must schedule the pod on the same node as the storage.
3. **Not Suitable for Multi-Node Access**: Unlike network-attached storage, local storage cannot be shared across multiple nodes.

Local storage is a powerful option in Kubernetes when used with the appropriate workloads and data durability considerations.

## ConfigMap and Secret

Can be accessed in ways
- Environment Variables: Injected into a container's environment.
- **Volumes**: Mounted as files inside a container.
- Command-line Arguments: Used as arguments when starting containers && Direct Reference: Used by other Kubernetes resources, such as image pull secrets.

In Kubernetes, **ConfigMaps** and **Secrets** are not volumes themselves, but they can be mounted into a pod as **volumes**.

### Mounting as Volumes
When you mount a ConfigMap or Secret as a volume:
1. You define it in the `volumes` section of a pod's specification.
2. You specify how and where to mount it in the container.

#### **ConfigMap Volume Example**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-volume-example
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: my-configmap
```

#### **Secret Volume Example**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-example
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
  volumes:
  - name: secret-volume
    secret:
      secretName: my-secret
```

In both cases, the volume is a way to deliver the ConfigMap or Secret data into the pod's filesystem.

## Components
- Persistent Volume
  - **cluster** resource
  - Can be created via yaml file
  - Abstraction that needs real physical storage, like
    - local hard drive/disk
    - nfs server
    - cloud storage
  - **Interface to the actual storage**
  - The storage is like external plugin to the cluster(storage backend)
  - **NOT namespaced**, accessible to the whole cluster
- Persistent Volume Claim
- Storage Class
