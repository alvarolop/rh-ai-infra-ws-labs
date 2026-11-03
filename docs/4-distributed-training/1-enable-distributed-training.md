# ⚙️ Enable Distributed Training Components

## Install DataScienceCluster components

Distributed training on OpenShift AI requires specific components in the `DataScienceCluster` (DSC). At minimum, for Kubeflow Trainer workloads from a workbench or the CLI:

| Component | Pipelines only | Workbenches only | Pipelines and workbenches |
| --- | --- | --- | --- |
| dashboard | Managed | Managed | Managed |
| aipipelines | Managed | Removed | Managed |
| kueue | Unmanaged | Unmanaged | Unmanaged |
| ray | Managed | Managed | Managed |
| trainingoperator | Managed | Managed | Managed |
| workbenches | Removed | Managed | Managed |

?> It is important to avoid defining kueue as *Managed* because we will install Red Hat Build of Kueue (RHBoK) 

1. As `cluster-admin`, open the OpenShift console and navigate to **Ecosystem** → **Operators** → **Installed Operators** → **Red Hat OpenShift AI** → **Data Science Cluster** → your DSC instance (for example, `default-dsc`).

2. Select the **YAML** tab and confirm the component states:

```yaml
spec:
  components:
    dashboard:
      managementState: Managed
    aipipelines:
      managementState: Managed
    kueue:
      managementState: Unmanaged
    ray:
      managementState: Managed
    trainingoperator:
      managementState: Managed
    workbenches:
      managementState: Managed
```

3. Save the DSC if you changed any values and wait for reconciliation to complete.

4. Open the OpenShift console and navigate to **Ecosystem** → **Operators** → **Software Catalog** → Filter by `Red Hat build of Kueue` →  Install the Operator

5. Wait for `Red Hat build of Kueue` operator installation to complete.

## Verify DataScienceCluster components

After changing DataScienceCluster and installing the `Red Hat build of Kueue` operator, the platform is ready to support any 

### Kueue Operator

Kueue queues and admits distributed workloads according to cluster quotas. Verify whether the OpenShift Kueue operator is deployed, and proceed with installation if it is not yet available:

1. Create a namespace for the Kueue Operator:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-kueue-operator
  labels:
    openshift.io/cluster-monitoring: "true"
EOF
```

2. Install the Kueue OperatorGroup in the namespace you created in the previous step by creating the following objects:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  generateName: openshift-kueue-operator-
  name: openshift-kueue-operator
  namespace: openshift-kueue-operator
spec: {}
EOF
```

3. Create the following Subscription CR:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: kueue-operator
  namespace: openshift-kueue-operator
spec:
  channel: "stable-v1.5"
  installPlanApproval: Automatic
  name: kueue-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

4. To verify that the Operator deployment is successful, run:

```bash
oc get pods -n openshift-kueue-operator
```

?> ✅ You should see pods in **Running** state.

```text
NAME                                      READY   STATUS    RESTARTS   AGE
openshift-kueue-operator-76bb977d5d-bl8t4   1/1     Running   0          28s
openshift-kueue-operator-76bb977d5d-sxw8s   1/1     Running   0          28s
```

Additionally, confirm that a Kueue instance has been created:

```bash
oc get kueue
```

* **Sample output** for the Kueue:

```text
NAME            READY   REASON
default-kueue   True    
```

### Verify Ray

In OpenShift AI, you can run a Ray-based distributed workload from a Jupyter notebook or from a pipeline. Confirm the Ray controller is running:

```bash
oc get pods -n redhat-ods-applications -l app.kubernetes.io/name=kuberay
```

* **Sample output** for the Kueue controller in `redhat-ods-applications`:

```text
NAME                               READY   STATUS    RESTARTS      AGE
kuberay-operator-8dddf9c89-gc2zp   1/1     Running   0             17h
```

## Verify Training Operator (Kubeflow Training Operator)

To reduce the time needed to train a Large Language Model (LLM), you can run the training job in parallel. In Red Hat OpenShift AI, the Kubeflow Training Operator and Kubeflow Training Operator Python Software Development Kit (Training Operator SDK) simplify the job configuration.

```bash
oc get pods -n redhat-ods-applications -l app.kubernetes.io/name=training-operator
```

* **Sample output** for the Training Operator in `redhat-ods-applications`:

```text
NAME                                          READY   STATUS    RESTARTS   AGE
kubeflow-training-operator-774b75cb67-s9qfw   1/1     Running   0          9m36s
```

## JobSet Operator

The **JobSet Operator** is an OpenShift and Kubernetes-native workload management solution designed to manage and orchestrate large-scale, coordinated batch computing workloads—specifically **distributed AI/ML model training** (such as PyTorch, JAX, or TensorFlow) and **High-Performance Computing (HPC)**.

The JobSet Operator is required in Red Hat OpenShift AI (RHOAI) because it serves as the underlying orchestration engine for Kubeflow Trainer v2 and Kueue, enabling multi-node distributed AI training jobs to execute with coordinated pod lifecycles, deterministic network discovery, and all-or-nothing failure recovery.

Verify whether the OpenShift JobSet operator is deployed, and proceed with installation if it is not yet available:

1. Create a namespace for the JobSet Operator:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-jobset-operator
  labels:
    name: openshift-jobset-operator
    openshift.io/cluster-monitoring: "true"
EOF
```

2. Install the JobSet OperatorGroup in the namespace you created in the previous step by creating the following objects:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  generateName: openshift-jobset-operator-
  name: openshift-jobset-operator
  namespace: openshift-jobset-operator
spec:
  targetNamespaces:
  - openshift-jobset-operator
EOF
```

3. Create the following Subscription CR:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: job-set
  namespace: openshift-jobset-operator
spec:
  channel: "stable-v1.0"
  installPlanApproval: Automatic
  name: job-set
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

4. To verify that the Operator deployment is successful, run:

```bash
oc get pods -n openshift-jobset-operator
```

?> ✅ You should see pods in **Running** state.

```text
NAME                                      READY   STATUS    RESTARTS   AGE
jobset-operator-854c7f58d8-hht5c   1/1     Running             0          3s
```

