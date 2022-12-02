# thesis-intro-presentation

## Free access to container filesystem vulnerability
- by default, containers in pods have read-write root filesystem and processes run under root
- in such filesystem, everybody with access to the container can tamper with files and read secret configuration, see system files, ...

### Demonstration
- create a scenario on killerconda - https://killercoda.com/playgrounds/scenario/kubernetes
- put the following contents to `/tmp/unsecure-pod.yml`:
```
apiVersion: v1
kind: Pod
metadata:
  name: unsecure-pod
spec:
  containers:
  - name: unsecure            
    image: ubuntu
    command: [ "sh", "-c", "sleep 1h" ]
```
- create a pod with `kubectl apply -f /tmp/unsecure-pod.yml`
- kubectl exec -it unsecure-pod  -- sh
- inside the container, you can modify configuration, read secret files, create new files...
- example: `touch a.txt` will create a new file

### Prevention
- put the following contents to `/tmp/more-secure-pod.yml`:
```
apiVersion: v1
kind: Pod
metadata:
  name: more-secure-pod
spec:
  # NOTE: ubuntu image doesn't have such user, correct way will be to use custom image that will inherit from ubuntu image and will create such user
  securityContext:
    runAsUser: 1000
  containers:
  - name: more-secure
    image: ubuntu
    command: [ "sh", "-c", "sleep 1h" ]
    securityContext:
      readOnlyRootFilesystem: true
```
- create a pod with: `kubectl apply -f /tmp/more-secure-pod.yml`
- access the pod: `kubectl exec -it more-secure-pod -- sh`
- with the current configuration of a security context, you no longer have root privileges and can't modify the root filesystem, and cannot see system files
- example: `touch a.txt` will result in - `touch: cannot touch 'a.txt': Read-only file system`

### Enforcing proper pod configuration with Kyverno
- create a scenario on killerconda - https://killercoda.com/playgrounds/scenario/kubernetes
- Install Kyverno (recomended way is through helm, but for demo it is faster with kubectl)
```
kubectl create -f https://raw.githubusercontent.com/kyverno/kyverno/main/config/install.yaml
```
- put the following contents to `/tmp/require-root-only-fs-policy.yml`:
```
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-ro-rootfs
  annotations:
    policies.kyverno.io/title: Require Read-Only Root Filesystem
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/severity: medium
    policies.kyverno.io/subject: Pod
    policies.kyverno.io/description: >-
      A read-only root file system helps to enforce an immutable infrastructure strategy;
      the container only needs to write on the mounted volume that persists the state.
      An immutable root filesystem can also prevent malicious binaries from writing to the
      host system. This policy validates that containers define a securityContext
      with `readOnlyRootFilesystem: true`.
spec:
  # Specifies action done when validation fails
  # Allowed values: 'Audit' (report but allow) | 'Enforce' (don't allow to create invalid resource)
  validationFailureAction: Enforce
  # Specifies if policy applies also on already existing resources
  # If set to true, then violations are reported (but existing resources are not blocked - 'audit' semantics)
  background: true
  rules:
  - name: validate-readOnlyRootFilesystem
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "Root filesystem must be read-only."
      pattern:
        spec:
          containers:
          - securityContext:
              readOnlyRootFilesystem: true
```
- create a cluster policy with: `kubectl apply -f /tmp/require-root-only-fs-policy.yml`
- to list cluster policies - `kubectl get cpol`
- try to create pod with writable root filesystem - ["demonstration" section](#demonstration)
```
kubectl apply -f /tmp/unsecure-pod.yml
Error from server: error when creating "/tmp/unsecure-pod.yml": admission webhook "validate.kyverno.svc-fail" denied the request: 

policy Pod/default/unsecure-pod for resource violation: 

require-ro-rootfs:
  validate-readOnlyRootFilesystem: 'validation error: Root filesystem must be read-only.
    rule validate-readOnlyRootFilesystem failed at path /spec/containers/0/securityContext/'
```
- to list policy reports: `kubectl get polr -A`
- try to create pod with read-only root filesystem from - ["prevention" section](#prevention)
  - pod will be sucessfully created

### Comparision with OPA Gatekeeper
- compare the readability (declarative vs imperative paradigm) of the same policy with OPA Gatekeeper
- https://open-policy-agent.github.io/gatekeeper-library/website/read-only-root-filesystem/