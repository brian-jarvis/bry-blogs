### Dynamic Admission Control
##### By: Brian Jarvis 

As an OpenShift administrator, do you require control of the creation of objects in a cluster?  Maybe your organization has rules around project naming, or resource limit specifications.  What if you needed to inject a sidecar container with every application pod?

In this article, I will look at how you can solve these requirements with a custom admission controller to perform validation and even change the object before it is created.

#### What is an Admission Controller?
[Admission controllers](https://docs.openshift.com/container-platform/4.5/architecture/admission-plug-ins.html) act as a gatekeeper that intercept API requests and are able to change the request object or deny its entry to the cluster altogether.  OpenShift has a number of admission controllers enabled by default such as the `LimitRanger` that mutates pods with default resource requests and limits. It also verifies that pods do not exceed the resource requirements specified in the `LimitRange` objects defined in a namespace.

Two of the most flexible admission controllers are the validating admission webhooks and mutating admission webhook controllers.  These controllers do not implement any policy decisions or mutation logic themselves.  Instead, they call out to a REST endpoint (a webhook).  By being implemented independent of the cluster, the webhook services can offer capabilities that surpass limitations of the cluster platform.  Admission controllers can accept, reject, or accept-with-modifications the object which is attempting to be created.

![API admission chain with mutating and validating admission plug-ins](./images/admissionflow.png)
*API admission chain with mutating and validating admission plug-ins*

Let's look at how to implement a mutating admission webhook to enable mutating the tolerations defined on a pod to control when pods are evicted from a node.  You can use a mutating webhook to validate and modify any object.  There are many scenarios where you might want to have a webhook mutate and validate an object such as:
- Verifying security settings, naming conventions, image usage
- Injecting labels, annotations, sidecar containers

#### Controlling Evictions
By default, OpenShift uses the Taint-Based Evictions feature to evict pods from a node that experience specific conditions, such as not-ready and unreachable.  During a node failure, OpenShift will automatically add taints to the node, and start evicting the pods to be rescheduled on another node.  During pod creation, an admission controller will automatically mutate the request to add tolerations for node failure conditions with a `tolerationSeconds: 300`.  This specifies the pod will not be evicted until the node has been tainted for five minutes. 
```yaml
spec
  tolerations:
    - key: node.kubernetes.io/not-ready
      operator: Exists
      effect: NoExecute
      tolerationSeconds: 300
    - key: node.kubernetes.io/unreachable
      operator: Exists
      effect: NoExecute
      tolerationSeconds: 300
```
For many organizations and application SLA policies, five minutes is too long a timespan to have a pod down.  

OpenShift does not have a configuration to change the default value, however with a simple `MutatingAdmissionWebhook` it is possible to modify the tolerations to a more acceptable value when the pod is created.

### The Mutating Webhook Server
All of the code from this sample is available at [brian-jarvis/ocp4-tolerations-mutating-webhook](https://github.com/brian-jarvis/ocp4-tolerations-mutating-webhook).  Complete deployment instructions are available with the code.

Now, I will highlight some of the more interesting details and demonstrate the webhook in action.

A webhook needs a simple TLS enabled HTTP server that adheres to the Kubernetes API.  Each request to the API Server is reviewed by the `MutatingWebhookConfiguration`. If the criteria is triggered, an `AdmissionReview` is set to the webhook server.  The `AdmissionReview` contains information about the request including the full definition of the object under review.

After processing by the webhook server, a response is generated consisting of an `AdmissionReview` object that contains an `AdmissionResponse`.  This consists of an Allowed & Results field which are filled with the admission decision and optional Patch to mutate the resource.

The web server needs to decide whether to admit the object and if any mutations should be applied.  In deciding if the object should be sent to the webhook service, the `MutatingWebhookConfiguration` uses two criteria; namespace selector and the resource operation's rules.

The server could further control which objects are mutated by using annotations.  In my example we will mutate all pods in the namespace, unless it has an annotation `mutator-webhook.bry/mutate: false` on the pod definition.
```golang
func mutationRequired(ignoredList []string, metadata *metav1.ObjectMeta) bool {
 // skip special kubernete system namespaces
 for _, namespace := range ignoredList {
   if metadata.Namespace == namespace {
     glog.Infof("Skip mutation for %v for it's in special namespace:%v", metadata.Name, metadata.Namespace)
     return false
   }
 }

 annotations := metadata.GetAnnotations()
 if annotations == nil {
   annotations = map[string]string{}
 }

 status := annotations[admissionWebhookAnnotationStatusKey]

 // determine whether to perform mutation based on annotation for the target resource
 var required bool
 if strings.ToLower(status) == "injected" {
   required = false
 } else {
   switch strings.ToLower(annotations[admissionWebhookAnnotationInjectKey]) {
   default:
     required = true
   case "n", "no", "false", "of":
     required = false
   }
 }

 glog.Infof("Mutation policy for %v/%v: status: %q required:%v", metadata.Namespace, metadata.Name, status, required)
 return required
}
```

For the webhook server to modify the request, it must return a json patch of the mutations that are to be applied.  It should not directly apply the mutations.
```golang
func addToleration(target, added []corev1.Toleration, basePath string) (patch []patchOperation) {
 <...>
   glog.Infof("Patch path %v", path)
   patch = append(patch, patchOperation{
     Op:    "add",
     Path:  path,
     Value: value,
   })
 }
 return patch
}

func createPatch(pod *corev1.Pod, mutateConfig *Config, annotations map[string]string) ([]byte, error) {
 var patch []patchOperation

 patch = append(patch, addToleration(pod.Spec.Tolerations, mutateConfig.Tolerations, "/spec/tolerations")...)
 <...>

 return json.Marshal(patch)
}

func (whsvr *WebhookServer) mutate(ar *v1beta1.AdmissionReview) *v1beta1.AdmissionResponse {
 req := ar.Request
 <...>
 patchBytes, err := createPatch(&pod, whsvr.mutateConfig, annotations)
 
 <...>
 glog.Infof("AdmissionResponse: patch=%v\n", string(patchBytes))
 return &v1beta1.AdmissionResponse{
   Allowed: true,
   Patch:   patchBytes,
   PatchType: func() *v1beta1.PatchType {
     pt := v1beta1.PatchTypeJSONPatch
     return &pt
   }(),
 }
}
```

#### Configuring OpenShift
The mutations the webhook server applies will be stored in a config map.  For this example I have configured the tolerations specific to node not-ready/unreachable events with a more acceptable 15 second toleration.
```yaml
metadata:
 name: tolerations-mutator-config
 namespace: mutating-webhook
data:
 mutation.yml: |
   tolerations:
     - key: node.kubernetes.io/not-ready
       operator: Exists
       effect: NoExecute
       tolerationSeconds: 15
     - key: node.kubernetes.io/unreachable
       operator: Exists
       effect: NoExecute
       tolerationSeconds: 15
```

To allow the OpenShift apiserver to communicate with the webhook, I have created an `APIService`.  This will configure our webhook server as an aggregated API server. It allows other OpenShift Container Platform components to communicate with the webhook using internal credentials and facilitates testing using the oc command. Additionally, this enables role based access control (RBAC) into the webhook and prevents token information from other API servers from being disclosed to the webhook.

Because the OpenShift API requires a TLS connection we need to identify the trust certificate used for the web service.  OpenShift 4.5 includes the ability to [inject the service signing cert CA bundle](https://docs.openshift.com/container-platform/4.5/security/certificates/service-serving-certificate.html#add-service-certificate-apiservice_service-serving-certificate) into the APIService by using the *inject-cabundle* annotation, .  This makes setting up a service trust an easy process.
```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
 name: v1beta1.admission.online.openshift.io
 annotations:
   service.beta.openshift.io/inject-cabundle: "true"
spec:
 group: admission.online.openshift.io
 groupPriorityMinimum: 1000
 versionPriority: 15
 service:
  namespace: mutating-webhook
  name: tolerations-mutator
  path: "/mutate"
 version: v1beta1
```

The `MutatingWebhookConfiguration` defines how and when the webhook gets triggered.  Here I specify the service as kubernetes so the call goes through the `APIService` defined above.  The rules will determine which objects and under which conditions the webhook is called.  In this case, the webhook will be executed when a pod is created or updated.  Finally, a *namespaceSelector* is used to limit the rules to only apply on objects contained in namespaces with the associated label.

As with the APIService, we also inject caBundle in the `MutatingWebhookConfiguration` using the same annotation for the service signing cert service.
```yaml
apiVersion: admissionregistration.k8s.io/v1beta1
kind: MutatingWebhookConfiguration
metadata:
 name: mutationservices.admission.online.openshift.io
 annotations:
   service.beta.openshift.io/inject-cabundle: "true"
webhooks:
- name: mutationservices.admission.online.openshift.io
 clientConfig:
   service:
     namespace: default
     name: kubernetes
     path: /apis/admission.online.openshift.io/v1beta1/mutationservices
 rules:
 - operations: ["CREATE", "UPDATE"]
   apiGroups: [""]
   apiVersions: ["v1"]
   resources: ["pods"]
 namespaceSelector:
   matchLabels:
     webhook.toleration-mutate: enabled
```

#### Verify Webhook
Once the webhook pods are running, we will run through a few simple tests to verify it is operating as expected before testing to mutate a pod. First ensure the health status of the APIService and the conditions show all checks passed and the service is available.
```bash
$ oc describe apiservice \
   v1beta1.admission.online.openshift.io
Name:         v1beta1.admission.online.openshift.io
Namespace:    
Labels:       <none>
Annotations:  service.beta.openshift.io/inject-cabundle: true
API Version:  apiregistration.k8s.io/v1
Kind:         APIService
<...>
Status:
  Conditions:
    Last Transition Time:  2020-09-28T15:33:21Z
    Message:               all checks passed
    Reason:                Passed
    Status:                True
    Type:                  Available
Events:                    <none>
```

With the APIService passing all checks, running `oc get` allows us to call the service through the OpenShift APIserver.
```bash
# an empty request should return empty json with a 200 code.
$ oc get --raw /apis/admission.online.openshift.io/v1beta1/mutationservices/mutate/
{}
```

Creating a pod should have the default tolerations.  Without the label to enable the webhook server, OpenShift should create the pod in the same manner as before the webhook is created.
```bash
$ oc create -f ./test/namespace.yml -f ./test/pod.yml
namespace/chewie-mutate created
pod/wookie-test created

$ oc get pod wookie-test -n chewie-mutate  -o yaml | \
    awk '/tolerations/,/volumes/'
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
```

As a final test, the label is applied to the namespace, and the pod is recreated.  The pod is now mutated and will only tolerate the node being not-ready/unreachable for fifteen seconds instead of the default five minutes.
```bash
$ oc label namespace chewie-mutate webhook.toleration-mutate=enabled
namespace/chewie-mutate labeled

$ oc delete  pod wookie-test -n chewie-mutate
pod "wookie-test" deleted

$ oc create -f ./test/pod.yml
pod/wookie-test created
$ oc get pod wookie-test -n chewie-mutate  -o yaml | awk '/tolerations/,/volumes/'
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 15
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 15
```

#### Summary
There are many use cases where using an `MutatingAdmissionWebhook` makes managing a cluster and applications easier.  We successfully solved the node Not-Ready delay toleration to meet our application SLA requirement.  Expanded use could include validating with CMCD configurations, injecting application integrations such as AppDynamics and Hashicorp Vault sidecar containers.


Original inspiration and code forked from: [](https://medium.com/ibm-cloud/diving-into-kubernetes-mutatingadmissionwebhook-6ef3c5695f74)

#### Additional Resources
 - https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/
 - https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook 
 - https://docs.openshift.com/container-platform/4.5/architecture/admission-plug-ins.html#admission-plug-ins-default_admission-plug-ins
 - https://docs.openshift.com/container-platform/4.5/nodes/scheduling/nodes-scheduler-taints-tolerations.html#nodes-scheduler-taints-tolerations-about-taintBasedEvictions_nodes-scheduler-taints-tolerations
 - https://www.openshift.com/blog/a-podpreset-based-webhook-admission-controller 

![](./images/test_plugin_zenburn.svg)

embded
<script id="asciicast-379637" src="https://asciinema.org/a/379637.js?" data-theme="solarized-dark" data-preload=1 async></script>
