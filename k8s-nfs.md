That's a fantastic idea\! A `README.md` is the ideal format for presenting the final, complete, and fully-corrected solution to your management or team.

Here is the entire configuration and corrected procedure in a comprehensive Markdown format.

-----

# 🌐 Jenkins on Kubernetes with Dynamic NFS Persistence

This guide details the complete, tested, and corrected configuration for deploying Jenkins with resilient, dynamic storage using a dedicated NFS server and provisioner in a Kubernetes (Kind) environment.

## 1\. Project Overview

| Component | Function | Status |
| :--- | :--- | :--- |
| **Jenkins** | CI/CD Server | **Running (Persistent)** |
| **Storage** | Dynamic Provisioning via NFS | **Stable** |
| **Environment** | Kubernetes (Kind) | **Local Cluster** |

-----

## 2\. Manifests (`.yaml` Files)

This section includes the complete YAML definitions for all core resources. These files should be applied to the `jenkins` namespace.

### 2.1. NFS Server and Service Manifest (`nfs-server.yaml`)

This deployment sets up the actual NFS server pod and a ClusterIP Service for internal networking.

```yaml
# Source: nfs-server.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-server-data-pvc
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard # Use the default storage for the NFS server's host volume
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-server
  namespace: jenkins
  labels:
    app: nfs-server
spec:
  selector:
    matchLabels:
      app: nfs-server
  replicas: 1
  template:
    metadata:
      labels:
        app: nfs-server
    spec:
      containers:
        - name: nfs-server
          image: gcr.io/google_containers/volume-nfs:0.8 # NFS server image
          ports:
            - name: nfs
              containerPort: 2049
            - name: mountd
              containerPort: 20048
            - name: rpcbind
              containerPort: 111
          securityContext:
            privileged: true
          volumeMounts:
            - name: nfs-data
              mountPath: /exports
      volumes:
        - name: nfs-data
          persistentVolumeClaim:
            claimName: nfs-server-data-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: nfs-server
  namespace: jenkins
spec:
  selector:
    app: nfs-server
  ports:
    - name: nfs
      port: 2049
      targetPort: 2049
    - name: mountd
      port: 20048
      targetPort: 20048
    - name: rpcbind
      port: 111
      targetPort: 111
  type: ClusterIP
```

### 2.2. NFS Provisioner and StorageClass Manifest (`nfs-provisioner.yaml`)

This sets up the dynamic provisioner logic and the `nfs-dynamic` StorageClass that Jenkins will request.

```yaml
# Source: nfs-provisioner.yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-dynamic
  namespace: jenkins
provisioner: example.com/nfs
reclaimPolicy: Retain # IMPORTANT: Retain data in the PV if PVC is deleted
volumeBindingMode: Immediate
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  namespace: jenkins
  labels:
    app: nfs-client-provisioner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-provisioner-sa # Assuming this SA exists or is created
      containers:
        - name: nfs-client-provisioner
          # CORRECTED IMAGE: Uses the modern, stable provisioner image
          image: k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2 
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: example.com/nfs
            # CRITICAL FIX: The NFS_SERVER value must be patched later to the ClusterIP
            - name: NFS_SERVER
              value: "nfs-server.jenkins.svc.cluster.local" 
            - name: NFS_PATH
              value: /exports
      volumes:
        - name: nfs-client-root
          hostPath:
            path: /mnt/data
            type: DirectoryOrCreate
```

### 2.3. Jenkins Deployment Manifest (`jenkins-deployment.yaml`)

This deploys the Jenkins application, requesting the `nfs-dynamic` storage class.

```yaml
# Source: jenkins-deployment.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-dynamic-pvc
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: nfs-dynamic
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
  labels:
    app: jenkins-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins-server
  template:
    metadata:
      labels:
        app: jenkins-server
    spec:
      serviceAccountName: jenkins-admin # Assuming this SA exists or is created
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts
          ports:
            - containerPort: 8080
              name: httpport
            - containerPort: 50000
              name: jnlpport
          resources:
            # CORRECTED RESOURCE LIMITS (Fix for Insufficient memory)
            limits:
              cpu: "1"
              memory: 1536Mi
            requests:
              cpu: 500m
              memory: 1Gi
          env:
            # CORRECTED JAVA_OPTS (Fix for resource contention)
            - name: JAVA_OPTS
              value: "-Xms512m -Xmx1024m -Djenkins.install.runSetupWizard=false"
          volumeMounts:
            - name: jenkins-data
              mountPath: /var/jenkins_home
      volumes:
        - name: jenkins-data
          persistentVolumeClaim:
            claimName: jenkins-dynamic-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service
  namespace: jenkins
spec:
  type: NodePort
  selector:
    app: jenkins-server
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
      nodePort: 32000
      name: httpport
    - protocol: TCP
      port: 50000
      targetPort: 50000
      nodePort: 31115
      name: jnlpport
```

-----

## 3\. Deployment and Troubleshooting Execution Flow

These steps assume the `jenkins` namespace exists.

### Step 3.1: Apply Initial Manifests

```bash
kubectl apply -f nfs-server.yaml -n jenkins
kubectl apply -f nfs-provisioner.yaml -n jenkins
kubectl apply -f jenkins-deployment.yaml -n jenkins
```

### Step 3.2: Critical Fixes and Initialization

This sequence resolves the DNS, resource, and provisioner image issues.

1.  **Stop Jenkins Temporarily**

    ```bash
    kubectl scale deployment jenkins --replicas=0 -n jenkins
    ```

2.  **Patch Provisioner with ClusterIP (Fix for DNS Failure)**

    ```bash
    NFS_IP=$(kubectl get svc nfs-server -n jenkins -o jsonpath='{.spec.clusterIP}')
    kubectl set env deployment/nfs-client-provisioner -n jenkins NFS_SERVER="$NFS_IP"
    ```

3.  **Ensure PVC is Clean (Force Re-creation)**

    ```bash
    kubectl delete pvc jenkins-dynamic-pvc -n jenkins --ignore-not-found
    sleep 5 # Wait for cleanup

    # Re-apply the PVC to trigger provisioning with the corrected IP
    kubectl apply -f - <<EOF
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: jenkins-dynamic-pvc
      namespace: jenkins
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi
      storageClassName: nfs-dynamic
    EOF
    ```

4.  **Start Jenkins and Verify**

    ```bash
    kubectl scale deployment jenkins --replicas=1 -n jenkins
    ```

-----

## 4\. Final Verification of Persistence

### 4.1. Cluster Status Check

```bash
kubectl get pvc,pod -n jenkins
```

**Expected Result:** All pods should be **`Running (1/1)`** and the `jenkins-dynamic-pvc` status should be **`Bound`**.

### 4.2. End-to-End Data Persistence Test

1.  **Access Jenkins:**

      * Get the node IP: `NODE_IP=$(kubectl get node kind-control-plane -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')`
      * Jenkins URL: `http://${NODE_IP}:32000`

2.  **Create Test Job:** Log in and create a Freestyle Project named **`test-job-nfs`**.

3.  **Confirm Data Write on NFS Server:**

    ```bash
    # Get required variables
    NFS_POD=$(kubectl get pods -n jenkins -l app=nfs-server -o jsonpath='{.items[0].metadata.name}')
    PV_NAME=$(kubectl get pvc jenkins-dynamic-pvc -n jenkins -o jsonpath='{.spec.volumeName}')

    # Verify the job folder exists on the NFS share
    kubectl exec -n jenkins "$NFS_POD" -- sh -c "ls -ld /exports/*/$PV_NAME/jobs/test-job-nfs"
    ```

    **Verification:** A successful output confirming the existence of the `test-job-nfs` directory validates that Jenkins is writing to the persistent NFS volume.

4.  **Test Restart Resilience (Optional):**

      * Shutdown: `kubectl scale deployment jenkins --replicas=0 -n jenkins`
      * Startup: `kubectl scale deployment jenkins --replicas=1 -n jenkins`
      * Re-check the UI: The **`test-job-nfs`** project should still be present, confirming the solution is resilient.
