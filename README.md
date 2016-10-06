# Introduction

This tutorial will demo a number of technologies. The requirements for completing each section will be introduced in the section. If you want to download and prepare all of the prerequisites before the talk please go through each section.

This presentation was given at LinuxCon 2016 in Berlin. There are [slides](https://speakerdeck.com/philips/coreos-a-tutorial-on-hyperscale-infrastructure).

## etcd Basics

**Pre-requisites**

- [etcd and etcdctl](https://github.com/coreos/etcd/releases/tag/v3.0.10) for your laptop

First, run `etcd` in a terminal window.

```
./etcd
...
```

Storing and retrieving values is done simply using the put/get subcommands.

```
export ETCDCTL_API=3
./etcdctl put foo bar
```

```
./etcdctl get foo
```

With the `-w` flag additional information can be found. Notice the "revision"? With etcd all keys are revisioned and you can use this revision number to get old values of keys, setup multi-key transactions, and view all changes since a certain time.

Using two differnt put calls we will create a couple of known revisions for the key `foo`.

```
./etcdctl put foo bar -w json
{"header":{"cluster_id":17237436991929493444,"member_id":9372538179322589801,"revision":9,"raft_term":2}}
```

```
./etcdctl put foo toronto -w json
{"header":{"cluster_id":17237436991929493444,"member_id":9372538179322589801,"revision":11,"raft_term":2}}
```

With the revision number etcd can "time-travel" and look at old values of `foo`:

```
./etcdctl get foo --rev 9
foo
bar
```

```
./etcdctl get foo --rev 11
foo
toronto
```

```
./etcdctl get foo -w json
{"header":{"cluster_id":17237436991929493444,"member_id":9372538179322589801,"revision":11,"raft_term":2},"kvs":[{"key":"Zm9v","create_revision":2,"mod_revision":11,"version":10,"value":"dG9yb250bw=="}],"count":1}
```

## etcd Clustering (optional)

This section is optional. You can learn similar lessons by visiting [play.etcd.io](http://play.etcd.io)

**Pre-requisites**

- A working [Go environment](https://golang.org/doc/install)
- Follow the upstream guide to [setup a local cluster](https://github.com/coreos/etcd/blob/master/Documentation/dev-guide/local_cluster.md#local-multi-member-cluster)


After setting up the pre-requisites a three node etcd cluster will be running.

Members of the cluster can be listed like this:

```
./etcdctl member list
8211f1d0f64f3269, started, infra1, http://127.0.0.1:12380, http://127.0.0.1:2379
91bc3c398fb3c146, started, infra2, http://127.0.0.1:22380, http://127.0.0.1:22379
fd422379fda50e48, started, infra3, http://127.0.0.1:32380, http://127.0.0.1:32379
```

## Building an Application (optional)

**Pre-Requisites**

- A working local Docker client (`brew install docker`)
- A VM to run Docker, recommend [minikube](https://tectonic.com/blog/minikube-and-rkt.html)

```
git clone https://github.com/philips/2016-LinuxCon-EU-CoreOS-A-Tutorial-on-Hyperscale-Infrastructure
cd guestbook/v1
eval $(minikube docker-env)
VERSION=v1 REGISTRY=quay.io/philips make build
VERSION=v1 REGISTRY=quay.io/philips make push
```

## Running a Single Container

**Pre-Requisites**
- Any CoreOS virtual machine and SSH session

Now that we have built the container it is easy to run on a virtual machine with rkt:

```
rkt fetch docker://quay.io/philips/guestbook:v1 --insecure-options=image
```

Now, one thing to note is that rkt does not have a daemon. So, we really on your system init system to monitor the process. To do that quickly under systemd do something like this:

```
systemd-run rkt fetch docker://quay.io/philips/guestbook:v1 --insecure-options=image
```

Or with docker:

```
docker run quay.io/philips/guestbook:v1 
```

## Debugging with Toolbox (optional)

**Pre-Requisites**
- Any CoreOS virtual machine and SSH session

With fewer pieces of software on the host what happens to debugging tools? CoreOS provides a quick solution with `toolbox`:

```
toolbox
```

The environment can be customized to run any container whether that is Debian, Ubuntu, Fedora, Arch, etc.


## Kubernetes Basics

**Pre-Requisites**

- [kubectl 1.3.6+](https://coreos.com/kubernetes/docs/latest/configure-kubectl.html) installed and in your path
- A Kubernetes cluster, recommend [minikube](https://tectonic.com/blog/minikube-and-rkt.html)

With a working Kubernetes cluster it is possible to proxy through to localhost for development without having to worry about auth:

```
kubectl proxy
```

From there the API becomes very accessible using well-known tools like curl:

```
curl 127.0.0.1:8001/api/v1/services
```

The equivalent of this rest API call is:

```
kubectl describes services
```

## Kubernetes App Deployments

First, lets run the app that was built earlier and pushed to quay.io.

```
kubectl run guestbook --image quay.io/philips/guestbook:v1 -l app=guestbook
```

Confirm that the application is running by selecting all things that have `app=guestbook`:

```
kubectl get pods -l app=guestbook
NAME                         READY     STATUS    RESTARTS   AGE
guestbook-2893398214-x04rm   1/1       Running   0          4m
```

Neat! Now, connect to our application by selecting that pod process and forwarding the port.

```
kubectl port-forward $(kubectl get pods -l app=guestbook -o template --template="{{range.items}}{{.metadata.name}}{{end}}") 3000:3000
```

Visit: http://localhost:3000

Success! Kubernetes is now able to manage running a container! Cleanup:

```
kubectl delete deployment guestbook
```

## Kubernetes App Failures

Setup the application to run again using the kubectl run subcommand.

```
kubectl run guestbook --image quay.io/philips/guestbook:v1 -l app=guestbook
kubectl get pods -l app=guestbook
```

Kubernetes will allow the app instance to be killed.

```
kubectl delete guestbook-2893398214-ikc58
```

But, it will drive the cluster state torwards a single running instance. Within a few seconds a replacement is launched:

```
kubectl get pods -l app=guestbook
``` 

The reason that the single pod was replaced is because the deployment, which we will discuss later, that is driving the guestbook application 

```
kubectl describe deployment guestbook
```

## Kubernetes App Scaling

```
kubectl scale deployment guestbook --replicas=3
```

- Third party controllers for complex applications

## Kubernetes Services

Earlier we used port forwarding to confirm the application was running. This is fun, but this isn't terribly useful as no one outside of the cluster can reach our application. Delete the deployment and lets try exposing a port:

```
kubectl delete deployment guestbook
kubectl run guestbook --image quay.io/philips/guestbook:v1 -l app=guestbook --expose --port 3000
```

Now the service will have a cluster IP that is routable to other nodes on the cluster.

```
kubectl describe service guestbook
```

Often, this isn't terribly useful as users workstation's rarely are on the same network/VPC/overlay/etc that the cluster is on. The 10.0.0.0/24 address isn't routable. But, the IPs of the nodes are. And by using a type of service called a "NodePort" a port on the nodes will forward to the service.

Assuming we setup our cluster at http://toronto.example.com:31512/ 

```
kubectl edit service guestbook
```

It would be much more convienent however if the cluster setup a real load balancer. This can be done by editing the type once more from "NodePort" to "LoadBalancer"

```
kubectl edit service guestbook
kubectl describe service guestbook
```

This time the service description will include a "LoadBalancer Ingress" that is a real LoadBalancer depending on the environment of the cluster.

Now, cleanup everything:

```
kubectl delete deployment guestbook
kubectl delete service guestbook
```

## More on Services

Port-forward cluster local DNS to your workstation.

```
kubectl port-forward --namespace=kube-system $( kubectl get pods --namespace=kube-system -l k8s-app=kube-dns -o template --template="{{range.items}}{{.metadata.name}}{{end}}") 5300:53
```

Try and grab the redis-master service powering our website:

```
dig +vc -p 5300 @127.0.0.1  redis-master.default.svc.cluster.local
redis-master.default.svc.cluster.local. 30 IN A 10.3.0.25
```

For more [network debugging tips see this page](https://github.com/coreos/docs/blob/master/kubernetes/network-troubleshooting.md).

## Using Kubernetes in a Development Workflow

**Pre-Requisites**

- A working [Go environment](https://golang.org/doc/install)
- A working redis cluster from above
- goreman or another Procfile runner (`go get github.com/mattn/goreman`) 

It is very useful to be able to hack on code locally while using services running the cluster. Let's walk through the development workflow for the Guestbook service.

First, this setup is really naive so it only works when there is only one slave. Scale the slave replica set down to 1.

```
kubectl scale rs redis-slave --replicas=1
```

Now, inside of the guestbook subdirectory of this repo there is a Procfile. When ran with `goreman start` it will forward the redis master and slave.

```
goreman start
```

At this point you now have the live cluster database forwarding locally. This can be confirmed by querying the redis database using the CLI tooling:

```
redis-cli -h 127.0.0.1 -p 6380 keys '*'
```

Now hacking on the application is easy, simply go into the v2 directory and run:

```
REDIS_SLAVE=localhost:6379 REDIS_MASTER=localhost:6380 go run main.go
```

Its the best of both worlds!
