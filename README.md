## start up
```
./a.out --kubeconfig=/Users/xuzeng/.kube/dev-config --alsologtostderr=true
```
## CRD
We can create CRD in k8s directly without any code deploying in k8s, 
the api server will just save it to etcd and support its basic operations, such as get, list, watch, create, update, patch, delete.

At this stage, CRDs are just configurations.

```
# create network CRD
$ kubectl apply -f crd/network.yaml
# create network resources
$ kubectl apply -f example/example-network.yaml
$ kubectl get network
NAME            AGE
example-network 8s
```

## build controller atop CRD
We can turn CRD into program scope
* in pkg/apis/samplecrd/v1/types.go, map yaml configuration to go struct
* in pkg/apis/samplecrd/v1/register.go, for data schema register, so as the generated code (see later section) known how to decode json/yaml/pb from api server into go struct or encode go struct back to api server.

So controllers are do things like this:
1. ListAndWatch the latest resources state (emmm configuration) from api server
2. diff the latest state from api server with the real state in cluster
   * if no difference, just return ok
   * if there are difference, reconcile the resources in cluster (do the really CRUD dirty work) to match the latest desired state

### generate code with k8s code-generator
Project tree before kubernetes code-generate
```
$ tree $GOPATH/src/github.com/phosae/k8s-controller-custom-resource-demo
.
├── controller.go
├── crd
│   └── network.yaml
├── example
│   └── example-network.yaml
├── main.go
└── pkg
    └── apis
        └── samplecrd
            ├── register.go
            └── v1
                ├── doc.go
                ├── register.go
                └── types.go
```
Using k8s.io/code-generator to generate code for our CRD...
```
# install k8s.io/code-generator
$ go get -u k8s.io/code-generator

{
# project path, code generator work directory
ROOT_PACKAGE="github.com/phosae/k8s-controller-custom-resource-demo"
# API Group
CUSTOM_RESOURCE_NAME="samplecrd"
# API Version
CUSTOM_RESOURCE_VERSION="v1"

cd $GOPATH/src
# generate code, pkg/client are target directory for client code, pkg/apis are definition directory
./k8s.io/code-generator/generate-groups.sh all "$ROOT_PACKAGE/pkg/client" "$ROOT_PACKAGE/pkg/apis" "$CUSTOM_RESOURCE_NAME:$CUSTOM_RESOURCE_VERSION"
}

Generating deepcopy funcs
Generating clientset for samplecrd:v1 at github.com/phosae/k8s-controller-custom-resource-demo/pkg/client/clientset
Generating listers for samplecrd:v1 at github.com/phosae/k8s-controller-custom-resource-demo/pkg/client/listers
Generating informers for samplecrd:v1 at github.com/phosae/k8s-controller-custom-resource-demo/pkg/client/informers
```
Project tree after kubernetes code-generate
```
$ tree .
.
├── README.md
├── controller.go
├── crd
│   └── network.yaml
├── example
│   └── example-network.yaml
├── main.go
└── pkg
    ├── apis
    │   └── samplecrd
    │       ├── register.go
    │       └── v1
    │           ├── doc.go
    │           ├── register.go
    │           ├── types.go
    │           └── zz_generated.deepcopy.go
    └── client
        ├── clientset
        ├── informers
        └── listers
```
As we can see, zz_generated.deepcopy.go under pkg/apis/samplecrd/v1 are DeepCopy codes for Network and NetworkList
* DeepCopyInto
* DeepCopy
* DeepCopyObject

Package clientset, informers and listers under client directory are client libraries that we can write CRD controller atop it.

### Write Controller
Let's write our controller to bring generated clientset, informers and lister together.
 
```go
kubeClient, err := kubernetes.NewForConfig(cfg)
networkClient, err := clientset.NewForConfig(cfg)
networkInformerFactory := informers.NewSharedInformerFactory(networkClient, time.Second*30)

controller := NewController(kubeClient, networkClient,
		networkInformerFactory.Samplecrd().V1().Networks())

go networkInformerFactory.Start(stopCh)
```
The clientset and lister are just some HTTP2 client code can do Request to api server. 

The informer will leverage lister and clientset to list CRDs from api server at start up, and continually watch its update (done by its reflector component). 
Moreover, informer also support resources cache and index.

![](./client-go-controller-interaction.jpeg)

Our custom controller just need to register some call back function on informer, like in this project

```go
networkInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: controller.enqueueNetwork,
    UpdateFunc: func(old, new interface{}) {
        oldNetwork := old.(*samplecrdv1.Network)
        newNetwork := new.(*samplecrdv1.Network)
        if oldNetwork.ResourceVersion == newNetwork.ResourceVersion {
            // Periodic resync will send update events for all known Networks.
            // Two different versions of the same Network will always have different RVs.
            return
        }
        controller.enqueueNetwork(new)
    },
    DeleteFunc: controller.enqueueNetworkForDelete,
})
```
And in callback, we use a queue to accept change items, and use a go routine to reconcile desired state and read state continually. 

See controller#processNextWorkItem and controller#syncHandler. 