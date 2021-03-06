# Qiskit playground operator

## Prereqs

Build with the [SDK operator](https://sdk.operatorframework.io/docs/building-operators/golang/tutorial/).
For installation, follow [Installation guide](https://sdk.operatorframework.io/docs/building-operators/golang/installation/)

## Development

Create project

````
operator-sdk init --domain ibm.com --repo github.io/blublinsky/qiskitplaygrounds --project-version=3
````
Install additional libraries
````
go get github.com/go-logr/logr@v0.3.0
go get github.com/onsi/ginkgo@v1.14.1
go get github.com/onsi/gomega@v1.10.2
go get k8s.io/api/core/v1@v0.19.2
go get k8s.io/api/apps/v1@v0.19.2
go get github.com/openshift/api/route/v1
````
Create APIs

````
operator-sdk create api --group qiskit --version v1alpha1 --kind QiskitPlayground --resource --controller
````
## Overall approach

Implementation is based on [Jupyter images](https://jupyter-docker-stacks.readthedocs.io/en/latest/using/selecting.html)
A simple [Dockerfile](image/Dockerfile) is based on `jupyter/scipy-notebook`:

Changing permission in this dockerfile is necessary to be able to use this image on OpenShift, [see](https://developers.redhat.com/blog/2020/10/26/adapting-docker-and-kubernetes-containers-to-run-on-red-hat-openshift-container-platform/), which runs the pod as a user defined by project.

You can build image with the following command (assuming that you are in the project root directory)

````
docker build -t qiskit/jupyter:0.1 image
````
There is also a prebuild image `qiskit/jupyter:0.1`

## CRD

The CRD takes the following parameters:

````
	Image string `json:"image,omitempty"`
	ImagePullPolicy apiv1.PullPolicy `json:"imagePullPolicy,omitempty"`
	PVC string `json:"pvc,omitempty"`
	LoadBalancer bool `json:"loadbalancer,omitempty" description:"Define if load balancer service type is supported. By default false"`
    Resources *apiv1.ResourceRequirements `json:"resources,omitempty"`

````
The only required parameter is image, for example:

````
apiVersion: "qiskit.ibm.com/v1alpha1"
kind: QiskitPlayground
metadata:
  name: test
  namespace: jupyter
spec:
  image: "qiskit/jupyter:0.1"
````
If not provided Image pull policy defaults to `IfNotPresent`. To change it add `imagePullPolicy: "Always"`

PVC is a PVC which will be used for persistence of notebooks, for example:

````
apiVersion: "qiskit.ibm.com/v1alpha1"
kind: QiskitPlayground
metadata:
  name: test
  namespace: jupyter
spec:
  image: "qiskit/jupyter:0.1"
  pvc: "claim-admin"
````
If PVC is not defined, an internal Pod disk is used. Note that without PVC all the notebooks will dissapear after the pod is deleted.
If a PVC is used, all notebooks are stored there and will survive redeployment.

Exposing operator UI, depends on where it runs - plain vanila kubernetes or openshift. Function [`func GetClusterType`](controllers/qiskitplayground_controller.go) is doing this based on
on discovery client. See this [article](https://developers.redhat.com/blog/2020/01/22/why-not-couple-an-operators-logic-to-a-specific-kubernetes-platform/) and also [here](https://developers.redhat.com/blog/2020/09/11/5-tips-for-developing-kubernetes-operators-with-the-new-operator-sdk/)
for details. If you use Fabric8 APIs, the same can be done using code similar to [this](https://github.com/fabric8io/kubernetes-client/blob/master/kubernetes-examples/src/main/java/io/fabric8/kubernetes/examples/CRDExample.java)

If running on OpenShift, a [route](https://docs.openshift.com/container-platform/4.7/rest_api/network_apis/route-route-openshift-io-v1.html) is created for Playground.  
If it is vanilla kubernetes, by default a [ClusterIP](https://rtfm.co.ua/en/kubernetes-clusterip-vs-nodeport-vs-loadbalancer-services-and-ingress-an-overview-with-examples/) type service is created, and in order to
expose a service to user, you need to run `port-forward` on this service or define an `Ingress` for it. This works for `MiniKube` or `Kind`
kubernetes servers. Many of the cloud provider based kubernetes, for example EKS, GKE and Azure support `LoadBalancer` type service. For this type of service, a corresponding load balancer will be created automatically and ingress creation is not required. 
Defining loadbalancer parameter in the CR will create a loadbalancer service instead of cluster IP

````
apiVersion: "qiskit.ibm.com/v1alpha1"
kind: QiskitPlayground
metadata:
  name: test
  namespace: jupyter
spec:
  image: "qiskit/jupyter:0.1"
  loadbalancer: true

````

Finally resource allow to specify resources for the pod, for example:
````
apiVersion: "qiskit.ibm.com/v1alpha1"
kind: QiskitPlayground
metadata:
  name: test
  namespace: jupyter
spec:
  image: "qiskit/jupyter:0.1"
  pvc: "claim-admin"
  resources:
    requests:
      memory: "64Mi"
      cpu: "250m"
    limits:
      memory: "128Mi"
      cpu: "500m"
````

## Implementation details

To make implementation more reliable, operator is using a deployment with 1 replica,
which means that if the pod is deleted, it will be brought back by deployment.

When playground CR is deleted, it will delete all of the associated resources based on the owner reference.

## Build and Run commands

To run operator locally, use the following command:
````
make install run
````
To build operator's docker file run

````
make docker-build docker-push IMG=qiskit/operator:0.1
````

With this image in place, you can install operator.
````
make deploy IMG=qiskit/operator:0.1
````
By default, a new namespace is created with name <project-name>-system - `qiskit-operator-system` in our case
To get all resources created by this deployment run:

````
kubectl get all -n qiskit-operator-system
````
You will see that in addition to deployment and pod, a service for metrics will be created:

````
qiskit-operator-controller-manager-metrics-service   ClusterIP   10.96.240.190   <none>        8443/TCP   6m41s
````

## OLM support

To deploy an operator with OLM follow instructions [here](https://sdk.operatorframework.io/docs/building-operators/golang/tutorial/#3-deploy-your-operator-with-olm).
More details are [here](https://sdk.operatorframework.io/docs/olm-integration/quickstart-bundle/)
First install OLM using the following command:

````
operator-sdk olm install
````
To verify OLM installation you can execute:

````
operator-sdk olm status
````
All resources should be installed.

Now you can run

````
make bundle
````
Answer additional questions during build

Once the Bundle is created it has to be build and pushed to the registry using the command:

````
make bundle-build bundle-push BUNDLE_IMG=qiskit/operator-olm:0.1
````

***Note:*** Bundle-push does not seem to exist in the Makefile, so I pushed the create docker file manually.

Once the bundle is created we can test it using the following command

````
operator-sdk run bundle  docker.io/qiskit/operator-olm:0.1
````

