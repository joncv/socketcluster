# SCC Guide

SCC is a collection of services which allow you to easily deploy and scale SocketCluster to any number of machines (almost any number, see Notes at the bottom of this page).
SCC is designed to scale linearly and is optimized for running on Kubernetes but it can be setup without using an orchestrator.

SCC is made up of the following services:

- socketcluster https://github.com/SocketCluster/socketcluster
- scc-broker https://github.com/SocketCluster/scc-broker
- scc-state https://github.com/SocketCluster/scc-state
- scc-ingress (Kubernetes only)

## How it works

The **socketcluster** service can be made up of any number of regular SocketCluster instances - The main difference between running **socketcluster** as a single instance vs running it as a cluster is
that in cluster mode, you need to point each **socketcluster** instance to a working `scc-state` (state server) instance. Note that when you run **socketcluster** in this mode and the state server goes down, the **socketcluster** instance will keep trying to reconnect to the state server (and you might get some `Socket hung Up` errors until the state server becomes available again).

The **scc-broker** service can be made up of any number of **scc-broker** instances - This is a special backend-only service which is designed to broker
messages between multiple frontend-facing **socketcluster** instances. All the pub/sub channels in your entire system will be sharded evenly across available **scc-broker** instances.
Just like with the **socketcluster** instances above, each **scc-broker** instance needs to point to a state server in order to work.

The **scc-state** service is made up of a single instance - Its job is to dispatch the state of the cluster to all interested services to allow them to reshard themselves. The **scc-state** instance will notify all frontend **socketcluster** instances whenever a new backend **scc-broker** joins the cluster. This allows **socketcluster** instances to rebalance their pub/sub channels evenly across available brokers whenever a new **scc-broker** instance joins the cluster.

An SCC setup across multiple hosts may look like this (though the quantity of each instance type is likely to vary; except for **scc-state** which only ever has one instance per cluster):

<img alt="SCC diagram" src="assets/scc-diagram.jpg" title="SCC diagram" />

## Running on Kubernetes

Running on Kubernetes (K8s) is easy; you just need to run all the `.yaml` files from the `kubernetes/` directory from the SocketCluster repo (https://github.com/SocketCluster/socketcluster/tree/master/kubernetes) using the `kubectl` command (one at a time):

```
kubectl create -f <service-deployment-or-ingress-definition.yaml>
```

By default, you should also add a TLS/SSL key and cert pair to your provider (Rancher has a page were you can just paste them in).
Or if you don't want to use a certificate (not recommended), you can just delete these lines from `scc-ingress.yaml` before you create it with `kubectl`:

```
  tls:
  - secretName: scc-tls-credentials
```

Note that the step above is crucial if you don't want to use TLS/SSL - Otherwise the ingress load balancer service will not show up on your Rancher control panel until you add some credentials with the name `scc-tls-credentials` to your Rancher control panel (See Infrastructure &gt; Certificates page).

### Running on Kubernetes with Baasil (for simple development and deployment)

Baasil is an open source CLI tool and Baasil.io is a hosted 'as-a-service' Rancher/Kubernetes control panel.
If you want to try SCC on K8s and you do not already have a K8s environment/cluster, the simplest way to get started is to sign up to a free trial account on http://baasil.io

Note that you can use the baasil CLI tool (https://www.npmjs.com/package/baasil) to deploy your SocketCluster service/app to any Rancher/Kubernetes environment, you just have to modify the `~/.kube/config` file on your local machine to hold the configs for your own Rancher control panel (instead of the one hosted on Baasil.io).
It is strongly recommended that you use Kubernetes with Rancher for consistency. See http://rancher.com/ for more details.

To deploy to your own Rancher/K8s cluster and to run your service/app locally inside containers, follow the guides linked from this page (under `Installation`): https://github.com/SocketCluster/baasil-cli

## Running using Node.js directly

You can also run SCC using only Node.js version >= 6.x.x.
For simplicity, we will show you how to run everything on your localhost (`127.0.0.1`), but in practice, you will need to change `127.0.0.1` to an appropriate IP, host name or domain name.

First, you need to download each of these repositories to your machine(s):

- `git clone https://github.com/SocketCluster/scc-broker`
- `git clone https://github.com/SocketCluster/scc-state`

Then inside each repo, you should run `npm install` without any arguments to install dependencies.

Then you need to setup a new SocketCluster project to use as your user-facing instance. For info about how to setup a SocketCluster project, see this page: http://socketcluster.io/#!/docs/getting-started

Once you have the two repos mentioned earlier and your SocketCluster project setup, you should launch the state server first by
going inside your local **scc-state** repo and then running the command:

```
node server
```

Next, to launch a broker, you should navigate to your **scc-broker** repo and run the command:

```
SCC_STATE_SERVER_HOST='127.0.0.1' SOCKETCLUSTER_SERVER_PORT='8888' node server
```

Finally, to run a frontend-facing SocketCluster instance, you can navigate to your socketcluster project directory and run:

```
SCC_STATE_SERVER_HOST='127.0.0.1' SOCKETCLUSTER_PORT='8000' node server
```

You can add a second frontend-facing server by running (this time running on port 8001):

```
SCC_STATE_SERVER_HOST='127.0.0.1' SOCKETCLUSTER_PORT='8001' node server
```
Now if you navigate to either `localhost:8000` or `localhost:8001` in your browser, you should see that your pub/sub channels are shared between the two **socketcluster** instances.

Note that you can provide additional environment variables to various instances to set custom post numbers, passwords etc...
For more info, you can look inside the code in the `server.js` file in each repo and see what `process.env` vars are used.

When running multiples instances of any service on the same machine, make sure that the ports don't clash  - Modify the `SOCKETCLUSTER_SERVER_PORT` or `SOCKETCLUSTER_PORT` environment variable for each instance to make sure that they are unique.

## Notes

You should only ever run a single **scc-state** per cluster - This is currently a single point of failure (we plan to improve it at some point).
For this reason, it is recommended that you run this instance inside your datacenter/AWS availability zone and do not expose it to the public internet.

The **scc-state** instance does not handle any pub/sub messages and so it should not affect scalability of your cluster (it will scale linearly).

Note that you can launch the services in any order you like but if your state server crashes, you may get `Socket hung up` errors on other instances (while they keep trying to reconnect).
