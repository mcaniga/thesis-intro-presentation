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
- TBD with Kyverno