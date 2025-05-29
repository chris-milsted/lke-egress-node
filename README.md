# Testing egress from a single Linode using Kuma

Some use cases like having a single IP address for a firewall, or simplifying the creation of SPF records may mean that you want all traffic from your LKE clusters to exit via a single node. In a future write up I will also look at trying to do this on a linode and extending the cluster so that the method to migrate an IP address between linodes can be used to keep an even more fixed address. For now I am using a node in a second node pool in LKE.

## Create an LKE cluster

I am going to create a simple cluster with 3 worker nodes for applications, and also add a second node pool with a single node to use as my egress gateway. You can of course set the replicas to more than one and have more nodes for HA use cases.

The creation of the cluster I am not going to document here as I just used to the GUI and clicked my way through it. I can then grab my cluster ID from the linode command line like this
```bash
$ linode-cli lke clusters-list
┌────────┬────────────────────────┬────────┬─────────────┬─────────────────────────────────┬──────┐
│ id     │ label                  │ region │ k8s_version │ control_plane.high_availability │ tier │
├────────┼────────────────────────┼────────┼─────────────┼─────────────────────────────────┼──────┤
│ 458197 │ single-outbound-egress │ gb-lon │ 1.32        │ False                           │      │
└────────┴────────────────────────┴────────┴─────────────┴─────────────────────────────────┴──────┘
```

I wanted to grab the cluster ID to use in the `pools-list` command to check the pools and nodes align with my `kubectl get nodes` output. If you look at the node ID and the LKE node name, you should see this is the case:

```bash
$ linode-cli lke pools-list 458197
┌────────┬───────────────┬─────────────────────┬───────────────────┬──────────────┐
│ id     │ type          │ nodes.id            │ nodes.instance_id │ nodes.status │
├────────┼───────────────┼─────────────────────┼───────────────────┼──────────────┤
│ 672433 │ g6-standard-6 │ 672433-06c58ffd0000 │ 77689618          │ ready        │
├────────┼───────────────┼─────────────────────┼───────────────────┼──────────────┤
│ 672433 │ g6-standard-6 │ 672433-07d4d7550000 │ 77689621          │ ready        │
├────────┼───────────────┼─────────────────────┼───────────────────┼──────────────┤
│ 672433 │ g6-standard-6 │ 672433-53b61a420000 │ 77689619          │ ready        │
├────────┼───────────────┼─────────────────────┼───────────────────┼──────────────┤
│ 672483 │ g6-standard-4 │ 672483-435686ae0000 │ 77691719          │ ready        │
└────────┴───────────────┴─────────────────────┴───────────────────┴──────────────┘

$ kubectl get nodes -o wide
NAME                            STATUS   ROLES    AGE   VERSION   INTERNAL-IP       EXTERNAL-IP      OS-IMAGE                         KERNEL-VERSION         CONTAINER-RUNTIME
lke458197-672433-06c58ffd0000   Ready    <none>   22h   v1.32.1   192.168.159.162   172.236.8.189    Debian GNU/Linux 12 (bookworm)   6.1.0-30-cloud-amd64   containerd://1.7.25
lke458197-672433-07d4d7550000   Ready    <none>   22h   v1.32.1   192.168.159.187   172.236.22.50    Debian GNU/Linux 12 (bookworm)   6.1.0-30-cloud-amd64   containerd://1.7.25
lke458197-672433-53b61a420000   Ready    <none>   22h   v1.32.1   192.168.159.186   172.236.22.41    Debian GNU/Linux 12 (bookworm)   6.1.0-30-cloud-amd64   containerd://1.7.25
lke458197-672483-435686ae0000   Ready    <none>   21h   v1.32.1   192.168.159.32    172.237.96.231   Debian GNU/Linux 12 (bookworm)   6.1.0-30-cloud-amd64   containerd://1.7.25
```

The next step is to add some labels to make the app and gateway land on the correct nodes. I am using a pair of simple labels which are `zone: app` and `zone: egress`. You can add them to the nodes with a command similar to the below.

```bash
$ kubectl label nodes lke458197-672433-06c58ffd0000 lke458197-672433-07d4d7550000 lke458197-672433-53b61a420000 zone=app
node/lke458197-672433-06c58ffd0000 labeled
node/lke458197-672433-07d4d7550000 labeled
node/lke458197-672433-53b61a420000 labeled

$ kubectl label nodes lke458197-672483-435686ae0000 zone=egress
node/lke458197-672483-435686ae0000 labeled
```

## Install kuma onto K8s nodes

The first step is to install into LKE, note that we are going to pass some additional values into the helm chart to make sure that egress is enabled and also to place the egress gateway onto nodes which have the specific label of `zone: egress` in my case. I do this by passing in the [values file](./values.yaml)

```bash 
helm repo add kuma https://kumahq.github.io/charts
helm repo update
helm install -f values.yaml --create-namespace --namespace kuma-system kuma kuma/kuma 
```
You can check that the values are correct with the following command

```bash
$ helm -n kuma-system get values kuma
USER-SUPPLIED VALUES:
egress:
  enabled: true
  nodeSelector:
    zone: egress
  replicas: 1
```

Before we go any further we can check that for a non-mesh enabled namespace like default, applications route out of the cluster via their local node IP address. I have a [deployment](./deployment.yaml) which just curls the ifconfig.me website and prints the address. The application is also setup to land on the `zone: app` only nodes and has anti-affinity and three replicas, so we should get three pods. The ipaddresses should match the application nodes External-IP from the `kubectl` output above.

```bash
$ kubectl create -f deployment.yaml -n default

$ kubectl logs --selector app=curl
Curling to detect IP...
IP: , 172.236.8.189
Curling to detect IP...
IP: , 172.236.22.41
Curling to detect IP...
IP: , 172.236.22.50
```

Now we can setup the mesh and configure a new namespace called `mesh-namespace` to deploy the pods into again and see what happens.

## Creating the mesh

When helm installed the mesh, it created the default mesh but without some settings we need for egress routing to work. Firstly we need to enable egress routing and secondly we need to enable TLS as the routing is based on SNI naming. I have created a new [mesh](./mesh.yaml) object so just delete the default one and create the new one:

```bash
$ kubectl delete mesh default
mesh.kuma.io "default" deleted

$ kubectl create -f mesh.yaml 
```

Once that has been created we can add in some more objects and the namespace with the mesh enabled, we are going to apply the files [namespace](./namespace.yaml), then [external service](./external-service.yaml) and then a [mesh traffic permission](./meshTrafficPermission.yaml).

NOTE that the mesh traffic permission is a very broad allow-all, you can refine this down as needed for your security requirements.

```bash 
kubectl create -f namespace.yaml 
kubectl create -f external-service.yaml
kubectl create -f meshTrafficPermission.yaml 
```

Now we can test our application once again, but this time lets launch it in the `mesh-namespace` and see what happens:

```bash
$ kubectl create -f deployment.yaml -n mesh-namespace

$ kubectl logs --selector app=curl -n mesh-namespace
Curling to detect IP...
IP: , 172.237.96.231
Curling to detect IP...
IP: , 172.237.96.231
Curling to detect IP...
IP: , 172.237.96.231
```

There we have it, all egress from a single node.



