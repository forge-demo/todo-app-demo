# TODO Application Sample

NOTE: This sample project is adapted from https://github.com/microsoft/mindaro/tree/master/samples/todo-app .

## Setup Steps

### 1. Install devkit

```
$ go install github.com/Azure/forge/devkit/cmd/devkit
```

This step will build the devkit client binary in `<path-to-go-path>/bin/devkit`.

### 2. Clone this repo

```
$ git clone https://github.com/forge-demo/todo-app-demo.git
$ cd todo-app-demo
```

### 3. Connect to a Kubernetes cluster and create a namespace

```
$ kubectl create ns devkit-todo-app-demo
namespace/devkit-todo-app-demo created
```

### 4. Deploy this todo-app demo

```
$ kubectl -n devkit-todo-app-demo apply -f deployment.yaml
...omitted output
```

Wait a couple of minutes to wait for pod start up:

```
$ kubectl -n devkit-todo-app-demo get pods
NAME                            READY   STATUS    RESTARTS   AGE
database-api-69b87c7bd8-rbhln   1/1     Running   0          51m
frontend-5865c6f8df-7k44s       1/1     Running   0          51m
stats-api-589b6b8444-f97rm      1/1     Running   0          51m
stats-cache-55494d9d48-ssrbq    1/1     Running   0          51m
stats-queue-5c9d6c5597-9dtdx    1/1     Running   0          51m
stats-worker-b745b6cd9-6ttn9    1/1     Running   0          51m
todos-db-698dc6d64-tkr7j        1/1     Running   0          51m
```

Then get the service IP for the frontend service:

```
$ kubectl -n devkit-todo-app-demo get svc frontend
NAME       TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)        AGE
frontend   LoadBalancer   10.0.42.147   <some-public-ip>   80:30994/TCP   51m
```

And open the frontend service in a browser with `http://<some-public-ip>`.
If you can see a todo app in the browser, this means the demo is working.

## Devkit Usage Steps

### 0. Setup devkit

This demo has been setup with the devkit settings in below:

- `.devkit/` directory: store the devkit configs
- `.devkit-root`: specifies the root of the project (we only want to sync this demo folder for now)

### 1. Spawn devkit environment

```
# NOTE: this command might take a while for initial run as we need to sync the whole folder to remote...
$ devkit go --namespace devkit-todo-app-demo
checking devkit pod status
devkit pod devkit-todo-app-demo/stats-api is ready
forwarding remote ports to local
preparing workspace
connecting to remote
devkit ➜ /workspace $ ls
README.md  database-api  deployment.yaml  devkit.md  frontend  images  license.md  stats-api  stats-api-svc-patch.yaml  stats-worker
```

If you can see output like above, which means now you have entered the devkit environment successfully.

This environment runs in the remote cluster as a pod, providing:

- builtin tool chains, based on your project language (nodejs in our example)
- shell access, where we can run commands as in local
- automatically code sync from local to remote: we will edit the code from local then run in remote

### 2. Start `stats-api`

```
devkit ➜ /workspace $ cd stats-api
devkit ➜ /workspace/stats-api $ npm run start
...omitted output

> todo-stats-api@1.0.0 start
> node server.js

Listening on port 3001
```

Now the process is up and listening on port 3000.

### 3. "Bridging" `stats-api` service

Now open another terminal and run the following command:

```
$ kubectl -n devkit-todo-app-demo get pods
NAME                            READY   STATUS    RESTARTS   AGE
database-api-69b87c7bd8-rbhln   1/1     Running   0          81m
frontend-5865c6f8df-7k44s       1/1     Running   0          81m
stats-api                       1/1     Running   0          8m58s
stats-api-589b6b8444-f97rm      1/1     Running   0          81m
stats-cache-55494d9d48-ssrbq    1/1     Running   0          81m
stats-queue-5c9d6c5597-9dtdx    1/1     Running   0          81m
stats-worker-b745b6cd9-6ttn9    1/1     Running   0          81m
todos-db-698dc6d64-tkr7j        1/1     Running   0          81m
```

We can see that, the devkit pod is running with name `stats-api`. This pod is running with labels:

```
$ kubectl -n devkit-todo-app-demo get pods stats-api -o json | jq '.metadata.labels'
{
  "devkit/component": "stats-api",
  "devkit/profile": "default"
}
```

By default, the in-cluster traffic will not be redirected to this pod.
We need to mutate the service to make it work:

```
$ kubectl -n devkit-todo-app-demo apply -f stats-api-svc-patch.yaml
```

This will redirect the in-cluster traffic to the `stats-api` pod.
Now, go back to the browser and click to the "stats" page: you should be able to see some numbers,
which means the requests have been served from the `stats-api` pod. We can also confirm with the devkit shell output:

```
...previous output
request for stats received with kubernetes-route-as header: undefined
sending back response: {"todosCreated":"2","todosCompleted":0,"todosDeleted":0}
```

### 4. Making code change

Now everything is ready and we can make some code change. For example, we can edit the `server.js` in local:

```diff
     var resp = {
         todosCreated: created || 0,
-        todosCompleted: completed || 0,
+        todosCompleted: completed || 42,
         todosDeleted: deleted || 0,
     };
```

Then we can restart the process in the devkit shell using `ctrl-c`, then rerun:

```
^C
devkit ➜ /workspace/stats-api $ npm run start
...omitted output
```

Now the process is using the updated code - we can verify by refreshing from browser, which should show the updated numbers
in both the page and the devkit shell:

```
sending back response: {"todosCreated":"2","todosCompleted":42,"todosDeleted":0}
```