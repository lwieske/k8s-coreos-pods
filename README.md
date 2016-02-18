# Vagrant Kubernetes Cluster based on Pod Provisioning

Flow: The initial **vagrant up** creates and configures the vms. The following
call to **fleet start** launches kubelet on each node, which in turn launch
kube-master and kube-minion pods from their formerly rsnced yaml. The
containers - based on the hyperkube docker image - find each other to make a
working kubernetes cluster.

```
vagrant up --provider virtualbox
```

```                                                                       
┌─────────────────────────────────────────────────────────────────────────────---────────────┐
│                                  10.10.10                                                  │
└─────────┬────────────┬────────────┬────────────---──┬────────────┬────────────┬────────────┤
          │            │            │                 │            │            │            │
       .11│         .12│        .13 │             .101│        .102│        .103│        .104│
          │            │            │                 │            │            │            │
     ┌────┤       ┌────┤       ┌────┤            ┌────┤       ┌────┤       ┌────┤       ┌────┤
     │eth1│       │eth1│       │eth1│            │eth1│       │eth1│       │eth1│       │eth1│
┌────┴────┤  ┌────┴────┤  ┌────┴────┤    ┌───────┴─-──┤  ┌────┴────┤  ┌────┴────┤  ┌────┴────┤
│         │  │         │  │         │    │            │  │         │  │         │  │         │
│ etcd-01 │  │ etcd-02 │  │ etcd-03 │    │ node-01    │  │ node-02 │  │ node-03 │  │ node-04 │
│         │  │         │  │         │    │            │  │         │  │         │  │         │
│─────────│  │─────────│  │─────────│    │────────────│  │─────────│  │─────────│  │─────────│
│         │  │         │  │         │    │ kube       │  │ kube    │  │ kube    │  │ kube    │
│         │  │         │  │         │    │ master     │  │         │  │         │  │         │
│         │  │         │  │         │    │            │  │ minion  │  │ minion  │  │ minion  │
│         │  │         │  │         │    │────────────│  │─────────│  │─────────│  │─────────│
│         │  │         │  │         │    │ pod        │  │ pod     │  │ pod     │  │ pod     │
│         │  │         │  │         │    │            │  │         │  │         │  │         │
│         │  │         │  │         │    │ hyperkube  │  │hyperkube│  │hyperkube│  │hyperkube│
│         │  │         │  │         │    │            │  │         │  │         │  │         │
│         │  │         │  │         │    │ apiserver  │  │ proxy   │  │ proxy   │  │ proxy   │
│         │  │         │  │         │    │ controller │  │         │  │         │  │         │
│         │  │         │  │         │    │ scheduler  │  │         │  │         │  │         │
│         │  │         │  │         │    │────────────│  │─────────│  │─────────│  │─────────│
│         │  │         │  │         │    │ kubelet    │  │ kubelet │  │ kubelet │  │ kubelet │
│         │  │         │  │         │    │────────────│  │─────────│  │─────────│  │─────────│
│         │  │         │  │         │    │ fleet      │  │ fleet   │  │ fleet   │  │ fleet   │
│─────────│  │─────────│  │─────────│    │────────────│  │─────────│  │─────────│  │─────────│
│ etcd    │  │ etcd    │  │ etcd    │    │ etcd       │  │ etcd    │  │ etcd    │  │ etcd    │
│         │  │         │  │         │    │            │  │         │  │         │  │         │
│ peer    │  │ peer    │  │ peer    │    │ proxy      │  │ proxy   │  │ proxy   │  │ proxy   │
└─────────┘  └─────────┘  └─────────┘    └────────────┘  └─────────┘  └─────────┘  └─────────┘
```

```
$ fleetctl --endpoint http://10.10.10.11:2379 list-machines
MACHINE		IP		METADATA
73b64ef0...	10.10.10.102	core=node,kube=minion
aac099ca...	10.10.10.103	core=node,kube=minion
e80687b8...	10.10.10.100	core=node,kube=master
edfd9bff...	10.10.10.101	core=node,kube=minion
$
```

```
$ fleetctl --endpoint http://10.10.10.11:2379 start kube/*.service
Unit kubelet-master.service inactive
Unit kubelet-minion.service
Triggered global unit kubelet-minion.service start
Unit kubelet-master.service launched on 00e76d60.../10.10.10.100
$
```

Wait about 5 minutes until you see that the apiserver is started.

```
$ kubectl --server http://10.10.10.100:8080 cluster-info
Kubernetes master is running at http://10.10.10.100:8080
```

```
$ kubectl --server http://10.10.10.100:8080 get nodes
NAME           LABELS                                STATUS    AGE
10.10.10.101   kubernetes.io/hostname=10.10.10.101   Ready     4m
10.10.10.102   kubernetes.io/hostname=10.10.10.102   Ready     4m
10.10.10.103   kubernetes.io/hostname=10.10.10.103   Ready     4m
$
```

```
$ kubectl --server http://10.10.10.100:8080 get pods
NAME                       READY     STATUS    RESTARTS   AGE
kube-master-node-01        3/3       Running   0          5m
kube-minion-10.10.10.101   1/1       Running   0          1m
kube-minion-10.10.10.102   1/1       Running   0          1m
kube-minion-10.10.10.103   1/1       Running   0          1m
$
```

### Configuration (Head of Vagrantfile)

```
members = {
  #  Name     #, CPU,  RAM, 1ST_IP
  'etcd' => [ 3,   1,  256,     11 ],
  'node' => [ 4,   2, 2048,    100 ],
}
PREFIX   = "10.10.10"
IP_RANGE = "10.11.0.0/16"
```
