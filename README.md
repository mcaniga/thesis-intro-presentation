# thesis-intro-presentation

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