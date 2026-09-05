# Safely Reducing Disk Usage in a Kind Kubernetes Cluster

This guide documents a safe procedure for reducing disk and resource usage in a local [Kind](https://kind.sigs.k8s.io/) Kubernetes cluster.

The main scenario covered here is:

* A Kind cluster has multiple nodes.
* One worker node is no longer necessary.
* The worker contains Kubernetes workloads.
* Docker is consuming significant disk space.
* We want to remove the worker and its Docker storage **without accidentally deleting application data, Docker volumes, or system files**.
* We also want to identify old unused Kind images and remove them safely.

> **Important:** Never blindly run `docker system prune`, `docker volume prune`, or delete files directly from `/var/lib/docker`. Always inspect first.

---

# 1. The Problem

A long-running Kind cluster can consume significant disk space.

For example, a cluster might look like:

```text
my-cluster-control-plane
my-cluster-worker
my-cluster-worker2
```

Each Kind node is actually a Docker container.

Conceptually:

```text
Host Linux
│
└── Docker
    │
    ├── my-cluster-control-plane
    │      └── Docker volume → /var
    │
    ├── my-cluster-worker
    │      └── Docker volume → /var
    │
    └── my-cluster-worker2
           └── Docker volume → /var
```

Therefore, removing a Kind worker can potentially recover several gigabytes of disk space because the worker's Docker `/var` volume contains the node's Kubernetes/container runtime data.

---

# 2. Golden Rule: Inspect Before Deleting

The most important principle is:

```text
INSPECT → IDENTIFY → VERIFY → DELETE → VERIFY AGAIN
```

Do not start with:

```bash
docker system prune
```

or:

```bash
docker volume prune
```

or:

```bash
rm -rf /var/lib/docker/*
```

These commands can remove data that you still need.

Instead, identify exactly what is consuming space.

---

# 3. Check Host Disk Usage

Start with the filesystem:

```bash
df -h /
```

Example:

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda4        79G   63G   11G  85% /
```

For a more precise view:

```bash
df -B1 /
```

This is useful when comparing disk usage before and after cleanup.

---

# 4. Find Large Top-Level Directories

Check where the disk space is going:

```bash
sudo du -xh --max-depth=1 / | sort -h
```

Then investigate large directories:

```bash
sudo du -xh --max-depth=1 /var | sort -h
```

For Docker:

```bash
sudo du -xh --max-depth=1 /var/lib/docker | sort -h
```

Do not manually delete files inside Docker's storage directory.

Docker manages that directory itself.

---

# 5. Inspect Docker Storage

Start with:

```bash
docker system df
```

Example:

```text
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          8         4         1.986GB   1.159GB
Containers      6         4         3.28MB    4.733kB
Local Volumes   23        5         23.8GB    441.9MB
Build Cache     36        0         1.538kB
```

This gives a high-level overview.

For detailed information:

```bash
docker system df -v
```

This is much more useful because it shows:

* Images
* Containers
* Volumes
* Image sharing
* Image unique size
* Volume size
* Volume attachment count

---

# 6. Understand Docker Image Storage

An image's displayed size is not necessarily the amount of disk space that will be recovered if you delete it.

For example:

```text
IMAGE              SIZE     SHARED SIZE    UNIQUE SIZE
kindest/node       935MB    345.5MB        589.6MB
```

This means:

```text
935 MB total
│
├── 345.5 MB shared with another image
│
└── 589.6 MB unique to this image
```

If the image is removed, Docker can only reclaim the layers that are no longer required.

Therefore:

> **SIZE ≠ actual reclaimable disk space**

Always inspect `docker system df -v`.

---

# 7. Check Kind Nodes

List the Kind nodes:

```bash
kind get nodes --name my-cluster
```

Example:

```text
my-cluster-control-plane
my-cluster-worker
my-cluster-worker2
```

Also check Kubernetes:

```bash
kubectl get nodes
```

Example:

```text
NAME                       STATUS   ROLES
my-cluster-control-plane   Ready    control-plane
my-cluster-worker          Ready
my-cluster-worker2         Ready
```

These two views are important because:

* `kind get nodes` shows the Kind/Docker topology.
* `kubectl get nodes` shows Kubernetes Node objects.

---

# 8. Determine Whether a Worker Can Be Removed

Before removing a worker, inspect the Pods running on it.

```bash
kubectl get pods -A -o wide
```

To focus on one node:

```bash
kubectl get pods -A -o wide | grep my-cluster-worker2
```

A more precise command is:

```bash
kubectl get pods -A \
  --field-selector spec.nodeName=my-cluster-worker2
```

This tells us exactly which Pods are currently assigned to the worker.

---

# 9. Check Pod Ownership

This is important because not every Pod can automatically be recreated.

Inspect ownership:

```bash
kubectl get pods -A \
  --field-selector spec.nodeName=my-cluster-worker2 \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,OWNER:.metadata.ownerReferences[0].kind/metadata.ownerReferences[0].name'
```

Typical results:

```text
Deployment/ReplicaSet
StatefulSet
DaemonSet
```

A Pod managed by a Deployment/ReplicaSet can normally be recreated somewhere else.

A standalone Pod with no owner is different.

Example:

```text
default/app1-pod
OWNER: <none>
```

If this Pod is deleted, Kubernetes will **not automatically recreate it**.

---

# 10. Check Persistent Storage Before Draining

Always check PVCs and PVs:

```bash
kubectl get pvc -A
```

and:

```bash
kubectl get pv
```

If there are PersistentVolumes, understand where the data is stored before removing a node.

For example:

```text
PVC → PV → StorageClass → physical storage
```

A node removal should never be performed blindly when stateful workloads are involved.

---

# 11. Check Scheduling Constraints

A workload might be configured to run only on a specific node.

Inspect:

```bash
kubectl get pod <pod-name> -n <namespace> -o yaml
```

Look for:

```yaml
nodeSelector:
```

```yaml
affinity:
```

```yaml
nodeAffinity:
```

Also check:

```yaml
tolerations:
```

A Pod with something like:

```yaml
nodeSelector:
  kubernetes.io/hostname: my-cluster-worker2
```

cannot simply move to another worker.

---

# 12. Cordoning the Worker

Once we determine the node can be removed, prevent new workloads from being scheduled there:

```bash
kubectl cordon my-cluster-worker2
```

Verify:

```bash
kubectl get nodes
```

The node should show:

```text
SchedulingDisabled
```

This is an important safety step.

The node is still running, but Kubernetes will not place new Pods there.

---

# 13. Perform a Dry-Run Drain

Before actually moving Pods, perform a server-side dry run:

```bash
kubectl drain my-cluster-worker2 \
  --ignore-daemonsets \
  --dry-run=server
```

This is extremely useful.

It may tell you that some Pods require:

```text
--delete-emptydir-data
```

and that a standalone Pod requires:

```text
--force
```

Do not immediately add every option blindly.

Understand why Kubernetes is requesting each option.

---

# 14. Backup Standalone Pods

If a standalone Pod must be removed, save its manifest first.

Example:

```bash
kubectl get pod app1-pod -o yaml > app1-pod-backup.yaml
```

Verify the backup:

```bash
ls -lh app1-pod-backup.yaml
```

Now the Pod definition can be recreated later if necessary.

---

# 15. Drain the Worker

Once the dry run has been understood and the important data is protected:

```bash
kubectl drain my-cluster-worker2 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force
```

What this does:

1. Prevents new workloads from being scheduled.
2. Evicts workloads managed by controllers.
3. Gives controllers an opportunity to recreate them on another node.
4. Ignores DaemonSet Pods.

DaemonSets are special.

For example:

```text
kindnet
kube-proxy
```

normally run once per node.

They should not be manually moved like ordinary application Pods.

---

# 16. Verify Workloads After the Drain

Check all Pods:

```bash
kubectl get pods -A -o wide
```

Confirm that application workloads have moved to the remaining worker.

Also check:

```bash
kubectl get nodes
```

and:

```bash
kubectl get daemonsets -A -o wide
```

For example, after going from three nodes to two nodes:

```text
kindnet      DESIRED=2 CURRENT=2 READY=2
kube-proxy   DESIRED=2 CURRENT=2 READY=2
```

This is expected.

---

# 17. Remove the Kind/Docker Worker

Kind does not provide a normal public command such as:

```bash
kind delete node
```

for removing one individual node from an existing cluster.

After the node has been drained, the specific Kind node can be removed through Docker:

```bash
docker rm -f -v my-cluster-worker2
```

The `-v` is important here because the Kind worker has an anonymous Docker volume containing its `/var` data.

Before doing this, inspect the node:

```bash
docker inspect my-cluster-worker2
```

Useful information:

```bash
docker inspect my-cluster-worker2 \
  --format 'Name={{.Name}}
Cluster={{index .Config.Labels "io.x-k8s.kind.cluster"}}
Role={{index .Config.Labels "io.x-k8s.kind.role"}}
Image={{.Config.Image}}
Status={{.State.Status}}'
```

---

# 18. Verify the Docker Node Was Removed

Check:

```bash
docker ps -a --filter "name=my-cluster"
```

The removed worker should no longer appear.

Check Kind:

```bash
kind get nodes --name my-cluster
```

Expected:

```text
my-cluster-control-plane
my-cluster-worker
```

---

# 19. Remove the Stale Kubernetes Node Object

After the Docker/Kind node disappears, Kubernetes may temporarily still have the old Node object.

For example:

```text
my-cluster-worker2   NotReady   SchedulingDisabled
```

This is now only a Kubernetes API object.

Delete it:

```bash
kubectl delete node my-cluster-worker2
```

Then:

```bash
kubectl get nodes
```

Expected:

```text
my-cluster-control-plane   Ready
my-cluster-worker          Ready
```

---

# 20. Verify No Pods Remain on the Removed Node

Run:

```bash
kubectl get pods -A \
  --field-selector spec.nodeName=my-cluster-worker2
```

Expected:

```text
No resources found
```

This is one of the most important final checks.

---

# 21. Verify the Old Docker Volume Was Removed

List Docker volumes:

```bash
docker volume ls
```

For detailed storage usage:

```bash
docker system df -v
```

Kind node volumes often look like long random hexadecimal names.

For example:

```text
05f31710...   14.03GB
28e9ce...      1.673GB
```

Do not assume an anonymous volume is unused just because it has a random name.

Identify its owner:

```bash
docker ps -a --filter volume=<VOLUME_NAME>
```

Then:

```bash
docker volume inspect <VOLUME_NAME>
```

---

# 22. Example: Identifying a Large Kind Volume

Suppose:

```text
05f31710...   14.03GB   LINKS=1
```

Check:

```bash
docker ps -a --filter volume=05f31710...
```

If it returns:

```text
my-cluster-worker
```

then inspect:

```bash
docker inspect my-cluster-worker \
  --format '{{range .Mounts}}{{println .Type .Name .Destination}}{{end}}'
```

If the result is:

```text
volume 05f31710... /var
```

we know:

```text
05f31710...
      │
      └── my-cluster-worker
             │
             └── /var
```

That volume is **required**.

Do not delete it.

---

# 23. Understand the Remaining Kind Storage

A two-node cluster may look like:

```text
Docker
│
├── my-cluster-control-plane
│     └── /var
│          └── Kind volume ~1.7GB
│
└── my-cluster-worker
      └── /var
           └── Kind volume ~14GB
```

These volumes are part of the active Kubernetes cluster.

Removing them would destroy the node's local Kubernetes/container runtime state.

---

# 24. Find Old Unused Kind Images

Kind images can remain after clusters are recreated or Kubernetes versions change.

List images:

```bash
docker images
```

Look for:

```text
kindest/node
```

For example:

```text
kindest/node   <none>   17b4349087dd   935MB
kindest/node   <none>   07acecbd6244   938MB
```

Do not immediately delete the `<none>` image.

First inspect it.

---

# 25. Inspect an Old Kind Image

```bash
docker image inspect 17b4349087dd \
  --format 'ID={{.Id}}
RepoTags={{json .RepoTags}}
RepoDigests={{json .RepoDigests}}
Created={{.Created}}
Size={{.Size}}'
```

Then check whether any containers use it:

```bash
docker ps -a --filter ancestor=17b4349087dd
```

If there are no containers, investigate further.

Use:

```bash
docker system df -v
```

Example:

```text
<none>   <none>   17b4349087dd   935MB   345.5MB   589.6MB   0
```

This tells us:

```text
Total:       935MB
Shared:      345.5MB
Unique:      589.6MB
Containers:  0
```

If it is confirmed to be an obsolete Kind image, it can be removed specifically:

```bash
docker image rm 17b4349087dd
```

---

# 26. Verify Image Cleanup

After removing the image:

```bash
docker images
```

Then:

```bash
docker system df
```

And finally:

```bash
df -h /
```

Do not expect the entire image size to necessarily disappear from disk because Docker image layers can be shared.

---

# 27. Docker Volumes: Be Extremely Careful

List volumes:

```bash
docker volume ls
```

Detailed usage:

```bash
docker system df -v
```

A volume with:

```text
LINKS=0
```

means that no current container is attached to it.

It does **not** automatically mean:

> "This volume is useless."

It may contain valuable application data.

For example:

```text
dev_prometheus_bloggyspace_dev_data
dev_grafana_bloggyspace_dev_data
dev_loki_dev_data
dev_bloggyspace_dev_db
```

These may contain:

* Prometheus metrics
* Grafana configuration/data
* Loki logs
* PostgreSQL/MySQL data
* Development application state

Therefore, inspect before deleting.

---

# 28. Useful Volume Investigation Commands

Find containers using a volume:

```bash
docker ps -a --filter volume=<VOLUME_NAME>
```

Inspect the volume:

```bash
docker volume inspect <VOLUME_NAME>
```

Check its size:

```bash
docker system df -v
```

If necessary, inspect the actual data:

```bash
sudo du -sh /var/lib/docker/volumes/<VOLUME_NAME>/_data
```

Do this only for investigation.

Do not manually delete files from the volume directory.

---

# 29. What We Should NOT Do

Avoid these commands unless you fully understand their consequences:

```bash
docker system prune
```

```bash
docker system prune -a
```

```bash
docker volume prune
```

```bash
docker image prune -a
```

```bash
sudo rm -rf /var/lib/docker/*
```

Why?

Because these can remove:

* Docker images
* stopped containers
* networks
* build cache
* unused volumes
* application databases
* monitoring data
* Kubernetes node storage

The safest approach is targeted cleanup.

---

# 30. Complete Verification Checklist

After removing a Kind worker:

### Kubernetes

```bash
kubectl get nodes
```

Expected:

```text
control-plane   Ready
worker          Ready
```

### Pods

```bash
kubectl get pods -A -o wide
```

All important workloads should be:

```text
Running
```

and assigned to the remaining nodes.

### Removed node

```bash
kubectl get pods -A \
  --field-selector spec.nodeName=my-cluster-worker2
```

Expected:

```text
No resources found
```

### DaemonSets

```bash
kubectl get daemonsets -A -o wide
```

DaemonSet desired/current counts should match the number of remaining eligible nodes.

### Kind

```bash
kind get nodes --name my-cluster
```

Expected:

```text
my-cluster-control-plane
my-cluster-worker
```

### Docker

```bash
docker ps -a --filter "name=my-cluster"
```

Only the remaining Kind nodes should appear.

### Docker storage

```bash
docker system df
```

### Host disk

```bash
df -h /
```

---

# 31. Recommended Cleanup Workflow

For future use, this is the condensed procedure:

```text
1. Check host disk
       ↓
   df -h /

2. Inspect Docker
       ↓
   docker system df
   docker system df -v

3. Check Kind nodes
       ↓
   kind get nodes --name my-cluster
   kubectl get nodes

4. Inspect workloads on target worker
       ↓
   kubectl get pods -A -o wide

5. Check PVC/PV
       ↓
   kubectl get pvc -A
   kubectl get pv

6. Check scheduling constraints
       ↓
   nodeSelector / affinity / tolerations

7. Cordon target worker
       ↓
   kubectl cordon <node>

8. Dry-run drain
       ↓
   kubectl drain <node> --ignore-daemonsets --dry-run=server

9. Backup standalone Pods if necessary
       ↓
   kubectl get pod <pod> -o yaml > backup.yaml

10. Drain
       ↓
   kubectl drain <node> \
     --ignore-daemonsets \
     --delete-emptydir-data \
     --force

11. Verify workloads
       ↓
   kubectl get pods -A -o wide

12. Remove Kind/Docker node
       ↓
   docker rm -f -v <node>

13. Verify Kind
       ↓
   kind get nodes --name my-cluster

14. Delete stale Kubernetes Node object
       ↓
   kubectl delete node <node>

15. Verify Kubernetes again
       ↓
   kubectl get nodes

16. Inspect Docker storage
       ↓
   docker volume ls
   docker system df -v

17. Remove only confirmed-unused old images
       ↓
   docker image rm <image>

18. Final disk check
       ↓
   df -h /
```

---

# 32. Example: What We Achieved

In this lab, the original cluster had:

```text
3 nodes

my-cluster-control-plane
my-cluster-worker
my-cluster-worker2
```

The target was to reduce it to:

```text
2 nodes

my-cluster-control-plane
my-cluster-worker
```

The worker was first cordoned and drained.

Application workloads successfully moved to the remaining worker.

The standalone Pod was identified as a special case and handled separately.

The Kind worker container was then removed:

```bash
docker rm -f -v my-cluster-worker2
```

The stale Kubernetes Node object was removed:

```bash
kubectl delete node my-cluster-worker2
```

The old Kind image was also identified as unused and removed:

```bash
docker image rm 17b4349087dd
```

The final host disk usage improved substantially.

The system went from approximately:

```text
~63 GB used
~11 GB available
```

to approximately:

```text
~56 GB used
~19 GB available
```

The important point is that this was achieved through **targeted cleanup rather than indiscriminate Docker pruning**.

---

# 33. Key Lessons

### Lesson 1 — Kind nodes are Docker containers

A Kind cluster is not using physical/VM nodes in the traditional sense.

```text
Kind node
    =
Docker container
```

Therefore Docker commands are relevant when managing Kind node storage.

---

### Lesson 2 — A Docker volume can be Kubernetes node storage

A volume such as:

```text
05f31710...
```

may look meaningless because it has a random name.

But if:

```text
05f31710...
        ↓
my-cluster-worker
        ↓
/var
```

then it is essential Kubernetes node storage.

Never delete it just because its name is unfamiliar.

---

### Lesson 3 — `LINKS=0` requires investigation

This:

```text
LINKS=0
```

means:

```text
No current container is attached
```

It does not necessarily mean:

```text
Safe to delete
```

The volume may contain valuable persistent application data.

---

### Lesson 4 — Drain before removing a node

Never remove a Kubernetes worker first.

The safe order is:

```text
cordon
   ↓
inspect
   ↓
dry-run drain
   ↓
backup special workloads
   ↓
drain
   ↓
verify
   ↓
remove node
```

---

### Lesson 5 — Stateful workloads require extra care

Always inspect:

```bash
kubectl get pvc -A
kubectl get pv
```

before removing a node.

A stateless Deployment is very different from a StatefulSet using persistent storage.

---

### Lesson 6 — Targeted deletion is safer than pruning

Prefer:

```bash
docker image rm <specific-image>
```

over:

```bash
docker image prune -a
```

when you know exactly which image is obsolete.

Likewise, never delete a volume until its ownership and purpose are understood.

---

# 34. Quick Commands Reference

### Disk

```bash
df -h /
sudo du -xh --max-depth=1 /var | sort -h
```

### Docker

```bash
docker system df
docker system df -v
docker ps -a
docker images
docker volume ls
```

### Kind

```bash
kind get nodes --name my-cluster
kind version
```

### Kubernetes

```bash
kubectl get nodes
kubectl get pods -A -o wide
kubectl get pvc -A
kubectl get pv
kubectl get daemonsets -A -o wide
```

### Node removal

```bash
kubectl cordon <node>

kubectl drain <node> \
  --ignore-daemonsets \
  --dry-run=server

kubectl drain <node> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force

docker rm -f -v <node>

kubectl delete node <node>
```

### Verify

```bash
kubectl get nodes
kubectl get pods -A -o wide
kind get nodes --name my-cluster
docker ps -a --filter "name=my-cluster"
docker system df
df -h /
```

---

# Conclusion

Reducing disk usage in a local Kubernetes environment should be treated as an infrastructure operation, not simply a cleanup operation.

The safest mindset is:

```text
                    ┌──────────────┐
                    │   DISK FULL  │
                    └──────┬───────┘
                           ↓
                    ┌──────────────┐
                    │    INSPECT   │
                    └──────┬───────┘
                           ↓
                    ┌──────────────┐
                    │  IDENTIFY    │
                    │  OWNERSHIP   │
                    └──────┬───────┘
                           ↓
                    ┌──────────────┐
                    │   VERIFY     │
                    │  SAFE TO     │
                    │   REMOVE?    │
                    └──────┬───────┘
                           ↓
                    ┌──────────────┐
                    │   DELETE     │
                    │ ONLY TARGET  │
                    └──────┬───────┘
                           ↓
                    ┌──────────────┐
                    │   VERIFY     │
                    │  EVERYTHING  │
                    │ STILL WORKS  │
                    └──────────────┘
```

**Never optimize storage by guessing.**

Inspect what owns the space, understand what it contains, remove only what is confirmed unnecessary, and verify both Kubernetes and the host afterward.

This approach is useful not only for Kind labs, but also as a general operational habit when troubleshooting disk pressure on Docker/Kubernetes development machines. 
> user@user-pc:~/projects/argocd_lab$ docker ps -a --filter volume=05f31710d8e8562bee637719ab88450d74759d8520d02b64fe8c0f3c7959d8da
> CONTAINER ID   IMAGE                  COMMAND                  CREATED        STATUS      PORTS     NAMES
2fa6e4fb0114   kindest/node:v1.34.3   "/usr/local/bin
