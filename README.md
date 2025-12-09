# Solution

Using `kind create cluster` for local environment.


## Terraform Problems

Initialized and applied Terraform and it was successful to create namespace resource.

![Terraform Success screenshot](<terraform_success.png>)

## Helm Chart Problems

1. Invalid container resource requests and limits
    - After running `helm template .` with a quick scan I see that container resource requests are invalid since requests > limits. 
    - Error if you tried to install it.
        ```bash
        * Deployment.apps "rc-app-rc-homework" is invalid: [spec.template.spec.containers[0].resources.requests: Invalid value: "512Mi": must be less than or equal to memory limit of 256Mi, spec.template.spec.containers[0].resources.requests: Invalid value: "500m": must be less than or equal to cpu limit of 250m]
        ```
    - Resolved by setting correctly in `values.yaml`

2. Hardcoded and incorrect namespace field for resources
    - In Deployment and Service resource, the `metadata.namespace` field was set to `homework`
    - Error if you tried to install it.
    ```bash
    Error: INSTALLATION FAILED: 2 errors occurred:
	* namespaces "homework" not found
	* namespaces "homework" not found
    ```
    - Resolved by using built in object `{{ .Release.Namespace }}` so namespace is set at install.

3. Service port not set, protocol invalid, and incorrect targetPort
    - 
    - Error if you tried to install it.
    ```bash
    * Service "rc-app-rc-homework" is invalid: [spec.ports[0].port: Invalid value: 0: must be between 1 and 65535, inclusive, spec.ports[0].protocol: Unsupported value: "tcp": supported values: "SCTP", "TCP", "UDP"]
    ```
    - Resolved by:
        - Change `service.protocol` value from `tcp` to `TCP`
        - Update `service.port` value to be `service.sourcePort` since that is the value name being used to set in service 
        - Change `service.targetPort` value from `8080` to `80` so it will match Deployments `containerPort`

4. Invalid nginx image tag
    - `helm install rc-app . -n rc-homework` will deploy successfully, but pod will not start because of `ImagePullBackOff` error.
    - Pod events will show problem
    ```bash
    k describe pod rc-app-rc-homework-84f9c7f655-g7x9m
    ...
    Failed to pull image "nginx:1.21-latest": rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/nginx:1.21-latest": failed to resolve reference "docker.io/library/nginx:1.21-latest": docker.io/library/nginx:1.21-latest: not found
    ```
    - Updated image from `1.21-latest` to `latest`. Looked at [Nginx Docker Images](https://hub.docker.com/_/nginx#image-variants) for a valid tag.


## Validation

Updated `~/.kube/config` with namespace default to rc-homework. Just to make my life a bit easier.

Terraform was successful, and namespace was created

```bash
k get ns rc-homework 
NAME          STATUS   AGE
rc-homework   Active   7h6m
```

`helm install rc-app .` deploys successfully. Resources are created and pod is running.

```bash
k get all
NAME                                      READY   STATUS    RESTARTS   AGE
pod/rc-app-rc-homework-6cf66cb48b-zwn4c   1/1     Running   0          6h22m

NAME                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/rc-app-rc-homework   ClusterIP   10.96.205.149   <none>        80/TCP    6h22m

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/rc-app-rc-homework   1/1     1            1           6h22m

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/rc-app-rc-homework-6cf66cb48b   1         1         1       6h22m
```

Verification that service works as expected. Service has endpoint equal to the pod IP. 

```bash
$ k get endpoints rc-app-rc-homework 
NAME                 ENDPOINTS       AGE
rc-app-rc-homework   10.244.0.6:80   6m22s

$ k get pods -o wide
NAME                                  READY   STATUS    RESTARTS   AGE     IP           NODE                 NOMINATED NODE   READINESS GATES
rc-app-rc-homework-6cf66cb48b-zwn4c   1/1     Running   0          7m16s   10.244.0.6   kind-control-plane   <none>           <none>
```

Service of type `clusterIP` can only be accessed from within the cluster. Exec into pod and `curl` the domain name for the service. The nginx homepage is returned which indicates it is working as expected. 
```bash
$ k exec -it rc-app-rc-homework-6cf66cb48b-zwn4c -- /bin/bash
root@rc-app-rc-homework-6cf66cb48b-zwn4c:/# curl http://rc-app-rc-homework.rc-homework.svc.cluster.local   
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Additionally we can use `kubectl port-forward` to make a temporary connection between our local machine and the service so we can access the application locally.
```bash
k port-forward service/rc-app-rc-homework 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
```

![Nginx Homepage Screenshot](<nginx_homepage.png>)
