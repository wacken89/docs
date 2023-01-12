### Get list of pods and nodes

**1.** Find all problem pods which not in `Running state`

```sh
kubectl get pods -A --field-selector=status.phase!=Running | grep -v Complete
```

!!! abstract "Output"

    ```
    ingress             traefik-ingress-controller-77b6cbdf77-m724s       0/2     Evicted            0          42d
    ingress             traefik-ingress-controller-77b6cbdf77-scsh5       0/2     Evicted            0          60d
    ingress             traefik-ingress-controller-77b6cbdf77-vn5c9       0/2     Evicted            0          79d
    testing             administration-v1-app-dd8699bb-znq6m              0/1     ImagePullBackOff   0          117m
    testing             administration-v1-app-ffff66c9c-gs8pw             0/1     ImagePullBackOff   0          99m
    testing             administration-v1-app-ffff66c9c-ngwjm             0/1     ImagePullBackOff   0          99m
    ```

**2.** Get list of nodes with ram capacity:

```sh
kubectl get no -o json | \
  jq -r '.items | sort_by(.status.capacity.memory)[]|[.metadata.name,.status.capacity.memory]| @tsv'
```

!!! abstract "Output"

    ```
    k8s-worker-system-01-dc14-fsn1	32171364Ki
    k8s-worker-system-01-dc3-nbg1	32171364Ki
    k8s-worker-system-02-dc14-fsn1	32171368Ki
    k8s-worker-system-02-dc3-nbg1	32171368Ki
    k8s-master-system-01-dc14-fsn1	7979256Ki
    k8s-master-system-01-dc3-nbg1	7979256Ki
    k8s-master-system-02-dc3-nbg1	7979264Ki
    ```

**3.** Get list of nodes with running pods on it:

```sh
kubectl get po -o json --all-namespaces | \
  jq '.items | group_by(.spec.nodeName) | map({"nodeName": .[0].spec.nodeName, "count": length}) | sort_by(.count)'
```

!!! abstract "Output"

    ```json
    [
      {
        "nodeName": "k8s-master-system-01-dc14-fsn1",
        "count": 7
      },
      {
        "nodeName": "k8s-master-system-01-dc3-nbg1",
        "count": 7
      },
      {
        "nodeName": "k8s-master-system-02-dc3-nbg1",
        "count": 7
      },
      {
        "nodeName": "k8s-worker-system-01-dc3-nbg1",
        "count": 14
      },
      {
        "nodeName": "k8s-worker-system-01-dc14-fsn1",
        "count": 18
      },
      {
        "nodeName": "k8s-worker-system-02-dc14-fsn1",
        "count": 18
      },
      {
        "nodeName": "k8s-worker-system-02-dc3-nbg1",
        "count": 22
      }
    ]
    ```


**3.**Script for list nodes which can not rollout `DaemonSet`

```sh
ns=my-namespace
pod_template=my-pod
kubectl get node | grep -v \"$(kubectl -n ${ns} get pod --all-namespaces -o wide | fgrep ${pod_template} | awk '{print $8}' | xargs -n 1 echo -n "\|" | sed 's/[[:space:]]*//g')\"
```


**4.** Get pod's that consume the maximum amount of cpu or memory:

```sh
# cpu
kubectl top pods -A | sort --reverse --key 3 --numeric

# memory
kubectl top pods -A | sort --reverse --key 4 --numeric
```

### Get other information

**1.** Debugging `Ingress` with `wide` output and finding labels:

```bash
kubectl -n jaeger get svc -o wide
```

!!! abstract "Output"

    ```
    NAME                            TYPE        CLUSTER-IP        EXTERNAL-IP   PORT(S)          AGE   SELECTOR

    jaeger-cassandra                ClusterIP   None              <none>        9042/TCP         77d   app=cassandracluster,cassandracluster=jaeger-cassandra,cluster=jaeger-cassandra
    ```

In this case, we immediately get a selector by which it finds the necessary pods.

**2.** Get limits and requests for each container of each pod:

```bash
kubectl get pods -n my-namespace -o=custom-columns='NAME:spec.containers[*].name,MEMREQ:spec.containers[*].resources.requests.memory,MEMLIM:spec.containers[*].resources.limits.memory,CPUREQ:spec.containers[*].resources.requests.cpu,CPULIM:spec.containers[*].resources.limits.cpu'
```

!!! abstract "Output"

    ```
    NAME                          MEMREQ   MEMLIM   CPUREQ   CPULIM
    sentry-web                    2Gi      2Gi      300m     700m
    sentry-web                    2Gi      2Gi      300m     700m
    sentry-web                    2Gi      2Gi      300m     700m
    sentry-web                    2Gi      2Gi      300m     700m
    sentry-worker                 1Gi      3Gi      300m     1
    sentry-worker                 1Gi      3Gi      300m     1
    sentry-worker                 1Gi      3Gi      300m     1
    sentry-worker                 1Gi      3Gi      300m     1
    sentry-worker                 1Gi      3Gi      300m     1
    ```

**3.** The kubectl run command (as well as create, apply, patch) has a wonderful ability to see changes before applying them - the `--dry-run` flag does this. And if used in conjunction with `-o yaml`, you can get the manifest of the required entity.

For example:

```bash
kubectl run test --image=grafana/grafana --dry-run -o yaml
```

!!! abstract "Output"

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      creationTimestamp: null
      labels:
        run: test
      name: test
    spec:
      replicas: 1
      selector:
        matchLabels:
          run: test
      strategy: {}
      template:
        metadata:
          creationTimestamp: null
          labels:
            run: test
        spec:
          containers:
          - image: grafana/grafana
            name: test
            resources: {}
    status: {}
    ```


**4.** Get an explanation on the manifest of a resource:

```bash
kubectl explain hpa
```

!!! abstract "Output"
    ```
    KIND:     HorizontalPodAutoscaler
    VERSION:  autoscaling/v1

    DESCRIPTION:
        configuration of a horizontal pod autoscaler.

    FIELDS:
      apiVersion    <string>
        APIVersion defines the versioned schema of this representation of an
        object. Servers should convert recognized schemas to the latest internal
        value, and may reject unrecognized values. More info:
        https://git.k8s.io/community/contributors/devel/api-conventions.md#resources

      kind    <string>
        Kind is a string value representing the REST resource this object
        represents. Servers may infer this from the endpoint the client submits
        requests to. Cannot be updated. In CamelCase. More info:
        https://git.k8s.io/community/contributors/devel/api-conventions.md#types-kinds

      metadata    <Object>
        Standard object metadata. More info:
        https://git.k8s.io/community/contributors/devel/api-conventions.md#metadata

      spec    <Object>
        behaviour of autoscaler. More info:
        https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status.

      status    <Object>
        current information about the autoscaler.
    ```


### Networks

**1.** Get internal IP addresses of cluster nodes:

```bash
kubectl get nodes -o json | \
  jq -r '.items[].status.addresses[]? | select (.type == "InternalIP") | .address' | \
  paste -sd "\n" -
```

!!! abstract "Output"
    ```
    49.12.46.61
    159.69.86.119
    159.69.30.239
    168.119.52.4
    159.69.158.121
    168.119.50.31
    116.203.153.163
    ```

**2.** List all services and the nodePort they occupy:

```bash
kubectl get --all-namespaces svc -o json | \
  jq -r '.items[] | [.metadata.name,([.spec.ports[].nodePort | tostring ] | join("|"))]| @tsv'
```

!!! abstract "Output"

    ```
    nginx-ingress-controller	32244|32245
    ```

**3**. In situations where there are problems with CNI, routes must be checked to identify the problem pod. This is where the pod subnets used in the cluster come in very handy:

```bash
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}' | tr " " "\n"
```

!!! abstract "Output"
    ```
    192.168.0.0/24
    192.168.1.0/24
    192.168.2.0/24
    192.168.4.0/24
    192.168.3.0/24
    192.168.6.0/24
    192.168.5.0/24
    ```


### Logs

**1.** Getting pod logs with human readable timestamp in case of its absence

```bash
kubectl -n my-namespace logs -f my-pod --timestamps
```

!!! abstract "Output"

    ```
    2020-07-08T14:01:59.581788788Z fail: Microsoft.EntityFrameworkCore.Query[10100]
    ```

**2.** Don't wait for the entire pod container log to be displayed - use `--tail`:

```bash
kubectl -n my-namespace logs -f my-pod --tail=50
```

**3.** Get logs from all pod containers:

```bash
kubectl -n my-namespace logs -f my-pod --all-containers
```

**4.** Get logs from all pods based on the label:

```bash
kubectl -n my-namespace logs -f -l app=nginx
```

**5.** Get the logs of the previous container, which, for example, fell:

```bash
kubectl -n my-namespace logs my-pod --previous
```
### List All Container Images Running in a Cluster

**1.** List all Container images in all namespaces

```bash
kubectl get pods --all-namespaces -o jsonpath="{..image}" |\
tr -s '[[:space:]]' '\n' |\
sort |\
uniq -c
```


```bash
kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec.containers[*].image}"
```

**2.** List Container images by Pod

```bash
kubectl get pods --all-namespaces -o=jsonpath='{range .items[*]}{"\n"}{.metadata.name}{":\t"}{range .spec.containers[*]}{.image}{", "}{end}{end}' |\
sort
```

**3.** List Container images filtering by Pod label

```bash
kubectl get pods --all-namespaces -o=jsonpath="{..image}" -l app=nginx
```

**4.** List Container images filtering by Pod namespace

```bash
kubectl get pods --namespace kube-system -o jsonpath="{..image}"
```

**5.** List Container images using a go-template instead of jsonpath 


```bash
kubectl get pods --all-namespaces -o go-template --template="{{range .items}}{{range .spec.containers}}{{.image}} {{end}}{{end}}"
```

### Other quick actions

**1.** How to copy all secrets from one namespace to another?

```bash
kubectl get secrets -o json --namespace namespace-old | \
  jq '.items[].metadata.namespace = "namespace-new"' | \
  kubectl create-f  -
```

**2.** Quickly create a self-signed certificate for tests:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=grafana.mysite.ru/O=MyOrganization"
kubectl -n myapp create secret tls selfsecret --key tls.key --cert tls.crt
```

**3.** Create job fron CronJob

```bash
kubectl create job --from=cronjob/<cronjob-name> <job-name>
```