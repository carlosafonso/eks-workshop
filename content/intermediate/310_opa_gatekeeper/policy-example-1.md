---
title: "Build Policy using Constraint & Constraint Template"
weight: 20
draft: false
---


#### 1. Build Constraint Templates

ConstraintTemplate describes the [Rego](https://www.openpolicyagent.org/docs/latest/#rego) that enforces the constraint and the schema of the constraint. The schema constraint allows the author of the constraint (cluster admin) to define the contraint behavior.

In this example, the cluster admin will force the use of unprivileged containers in the cluseter. The OPA Gatekeeper will look for the securitycontext field and check if `privileged=true`. If it's the case, then, the request will fail.

```bash
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8spspprivilegedcontainer
spec:
  crd:
    spec:
      names:
        kind: K8sPSPPrivilegedContainer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8spspprivileged

        violation[{"msg": msg, "details": {}}] {
            c := input_containers[_]
            c.securityContext.privileged
            msg := sprintf("Privileged container is not allowed: %v, securityContext: %v", [c.name, c.securityContext])
        }

        input_containers[c] {
            c := input.review.object.spec.containers[_]
        }

        input_containers[c] {
            c := input.review.object.spec.initContainers[_]
        }

```

Create it

```bash
kubectl create -f constrainttemplate.yaml

```

#### 2. Build Constraint

The cluster admin will use the constraint to inform the OPA Gatekeeper to enforce the policy. 
For our example, as cluster admin we want to enforce that all the created pod should not be privileged.

```bash
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPPrivilegedContainer
metadata:
  name: psp-privileged-container
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]

```

Create it

```bash
kubectl create -f constraint.yaml

```


#### 3. Test it
First, check for the CRD constraint and constrainttemplate were created.

```bash
kubectl get constraint
NAME                       AGE
psp-privileged-container   47s

kubectl get constrainttemplate
NAME                        AGE
k8spspprivilegedcontainer   60s

```

Second, let's try deploy a privileged nginx pod:
```bash
cat example.yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-nginx
  labels:
    app: bad-nginx
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      privileged: true

kubectl create -f example.yaml
Error from server ([denied by psp-privileged-container] Privileged container is not allowed: nginx, securityContext: {"privileged": true}): error when creating "example.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [denied by psp-privileged-container] Privileged container is not allowed: nginx, securityContext: {"privileged": true}

```

The requet was denied by kubernetes API, because it didn't meet the requirement from the constraint forced by OPA Gatekeeper.
