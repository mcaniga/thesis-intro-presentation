# thesis-intro-presentation

## Free access to container filesystem vulnerability
- by default, containers in pods have read-write root filesystem and processes run under root
- in such filesystem, everybody with access to the container can tamper with files and read secret configuration, see system files, ...

### Demonstration
- create scenario on killerconda - https://killercoda.com/playgrounds/scenario/kubernetes
- put following contents to `/tmp/unsecure-pod.yml`:
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
- create pod with `kubectl apply -f /tmp/unsecure-pod.yml`
- kubectl exec -it unsecure-pod  -- sh
- inside the container, you can modify configuration, read secret files, create new files...
- example: `touch a.txt` will create new file

### Prevention
- put following contents to `/tmp/more-secure-pod.yml`:
```
apiVersion: v1
kind: Pod
metadata:
  name: more-secure-pod
spec:
  # NOTE: ubuntu image don't have such user, correct way will be to use custom image that will inherit from ubuntu image and will create such user
  securityContext:
    runAsUser: 1000
  containers:
  - name: more-secure
    image: ubuntu
    command: [ "sh", "-c", "sleep 1h" ]
    securityContext:
      readOnlyRootFilesystem: true
```
- create pod with: `kubectl apply -f /tmp/more-secure-pod.yml`
- access the pod: `kubectl exec -it more-secure-pod -- sh`
- with current configuration of security context, you no longer have root priviledges and can't modify root filesystem, and cannot see system files
- example: `touch a.txt` will result in - `touch: cannot touch 'a.txt': Read-only file system`

## Unlimited resources vulnerability
- containers can be run with unlimited resources, which can cause the consumption of all host resources
- malicious container can execute a fork bomb, which will cause a denial of service on the host
- fork bomb explanation: https://www.cyberciti.biz/faq/understanding-bash-fork-bomb/

### Demonstration
- **!!!WARNING: Don't execute this example on your machine, use following commands on killerconda enviroment only!!!**
- create scenario on killerconda - https://killercoda.com/playgrounds/scenario/kubernetes
- open two tabs
- launch an insecure container in the first tab:
```
docker run --name unlimited --rm -it ubuntu bash
```
- monitor resource consumption of the container in second tab:
```
docker stats unlimited
```
- execute fork bomb in container (first tab)
```
:(){ :|:& };:
```
- in tab1, you can now see: "bash: fork: Resource temporarily unavailible"
- try to open new tab, it will be frozen
- to get new killercoda enviroment - click Restart

### Prevention
- launch container with resource constraints
```
docker run --name limited --rm -m 1Gi --cpus 0.6 -it ubuntu bash  
```
- now the fork bomb will not drain host resources

### Ensuring that ALL containers are executed with limited resources
- TBD with Kyverno