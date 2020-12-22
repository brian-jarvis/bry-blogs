### Expanding oc functionality with plugins
##### By: Brian Jarvis 

The oc/kubectl application has many useful functions. But what happens when you wish it could do something currently not available? 

[Plugins](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/) extend oc/kubectl with new sub-commands allowing cluster administrators to create more complex behavior for interacting with their cluster.

### Plugin Background
The Kubernetes [CLI special interest group](https://github.com/kubernetes/community/tree/master/sig-cli) added a built-in plugin system to kubectl that allows anyone to add new sub-commands. This does not require editing kubectl’s source code or recompiling it.

Because oc is built on top of kubectl it benefits from the same plugin system. This allows us to use the larger Kubernetes community of plugins available with our OpenShift clusters.

Any executable file in your PATH that starts with kubectl- or oc- can be called with the oc command. A plugin can be as simple as a bash scripts or a more complex applications written in a compiled language like Golang.

Let's look at creating a simple HelloWorld plugin to see how plugins work.  
https://asciinema.org/a/380652

![Krew](./images/krew-icon.png)  
Discovering, installing, updating plugins manually can be a tedious task. The kubernetes community developed [Krew](https://krew.sigs.k8s.io/) to make it easy to discover and manage plugins. Krew also maintains a centralized index of the community plugins available to be installed. There are currently ~120 plugins distributed through Krew.

### Getting started with Krew
Before we can use Krew we need to follow the [simple install process](https://krew.sigs.k8s.io/docs/user-guide/setup/install/). This will download the Krew plugin and place it in the correct location for use by oc.  
https://asciinema.org/a/380654

### Useful plugins
Now that we have installed Krew, let's look at a few of my favorite plugins available. You can find plugins either by running `oc krew search` or by searching the [Krew index](https://rew.sigs.k8s.io/plugins/).  

#### neat
When you run `oc get -o yaml` the output includes many attributes that make it difficult to view the important parts. This was made worse in OpenShift 4.5 with the introduction of the *metadata.managedFields* attribute which contains alot of extra properties. [Neat]() cleans up these extra properties and other fields that contain default values and cluter the output. Note it also removes the status field.
```
oc krew install neat
```
https://asciinema.org/a/380687

Neat removed over 250 lines from the console deployment yaml output. You can see in the above example the clean output neat provides. This is useful when reviewing the configuration of an object.

#### status

> get-all
> pod events/sick-pods
> rbac-lookup
> ksniff

#### Conclusion