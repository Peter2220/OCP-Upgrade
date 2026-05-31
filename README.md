# Root Cause Analysis (RCA)

## Issue Summary

During the OpenShift upgrade from **4.19.31** to **4.20.22**, the upgrade became stalled at approximately **78% completion**.

### Symptoms

ClusterVersion reported:

```text
Working towards 4.20.22:
755 of 959 done (78% complete),
waiting on node-tuning over 30 minutes which is longer than expected
```

The `node-tuning` ClusterOperator itself appeared healthy:

```text
Available=True
Progressing=False
Degraded=False
```

However, the upgrade process remained blocked waiting for the Node Tuning Operator.

---

## Investigation

### Cluster Version Status

```bash
oc get clusterversion
```

Output:

```text
Working towards 4.20.22:
755 of 959 done (78% complete),
waiting on node-tuning
```

### Node Tuning Operator Deployment

```bash
oc describe deploy cluster-node-tuning-operator \
-n openshift-cluster-node-tuning-operator
```

Observed:

```text
Replicas: 1 desired | 1 updated | 1 total | 0 available | 1 unavailable
```

Deployment condition:

```text
Available=False
Reason=MinimumReplicasUnavailable
```

This indicated that the deployment could not make its pod available.

---

### Pod Scheduling Failure

Examining the operator pod events:

```bash
oc describe pod \
-l name=cluster-node-tuning-operator \
-n openshift-cluster-node-tuning-operator
```

revealed:

```text
FailedScheduling

0/8 nodes are available:
8 node(s) didn't match Pod's node affinity/selector.
```

The deployment was configured with the following node selectors:

```yaml
nodeSelector:
  node-role.kubernetes.io/control-plane: ""
  node-role.kubernetes.io/master: ""
```

Therefore, the pod could only run on nodes containing both labels.

---

### Node Label Verification

Current node labels were reviewed and it was determined that the control plane nodes only had:

```text
node-role.kubernetes.io/master
```

and were missing:

```text
node-role.kubernetes.io/control-plane
```

As a result, the deployment selector requirements could not be satisfied and the operator pod remained unscheduled.

Example cluster node state before remediation:

```text
master1   Ready   master
master2   Ready   master
master3   Ready   master
worker1   Ready   worker
worker2   Ready   worker
worker3   Ready   worker
worker4   Ready   worker
worker5   Ready   worker
```

---

## Root Cause

The OpenShift control-plane nodes were missing the required label:

```text
node-role.kubernetes.io/control-plane
```

The upgraded version of the Node Tuning Operator deployment required both labels:

```text
node-role.kubernetes.io/master
node-role.kubernetes.io/control-plane
```

Because the `control-plane` label was absent, the operator pod could not be scheduled on any node.

This caused:

* Node Tuning Operator deployment to remain unavailable.
* ClusterVersion upgrade to wait indefinitely on the `node-tuning` operator.
* Upgrade progress to stall at approximately 78%.

---

# Resolution

The missing control-plane label was added to all master nodes.

```bash
oc label nodes \
-l node-role.kubernetes.io/master \
node-role.kubernetes.io/control-plane="" \
--overwrite
```

Result:

```text
node/master1 labeled
node/master2 labeled
node/master3 labeled
```

Verification:

```bash
oc get nodes
```

Output:

```text
master1   Ready   control-plane,master
master2   Ready   control-plane,master
master3   Ready   control-plane,master
worker1   Ready   worker
worker2   Ready   worker
worker3   Ready   worker
worker4   Ready   worker
worker5   Ready   worker
```

---

## Validation

After applying the labels:

1. The Node Tuning Operator pod was successfully scheduled.
2. The deployment became available.
3. The ClusterVersion Operator resumed processing the upgrade.

ClusterVersion status progressed from:

```text
755 of 959 done (78%)
waiting on node-tuning
```

to:

```text
763 of 959 done (79%)
waiting on olm
```

This confirmed that the Node Tuning Operator blockage had been resolved and the upgrade workflow continued normally.

---

# Conclusion

The OpenShift upgrade stalled because the Node Tuning Operator deployment required scheduling on nodes labeled with both:

```text
node-role.kubernetes.io/master
node-role.kubernetes.io/control-plane
```

The control-plane label was missing from all master nodes, preventing the operator pod from being scheduled. Adding the missing label restored operator availability and allowed the cluster upgrade to continue.
