### Expanding oc functionality with plugins
##### By: Brian Jarvis 

The oc/kubectl application has many useful functions.  But what happens when you wish it could do something currently not available?  

[Plugins](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/) extend oc/kubectl with new sub-commands allowing cluster administrators to create more complex behavior for interacting with their cluster.

#### Plugin Background
The Kubernetes [CLI special interest group](https://github.com/kubernetes/community/tree/master/sig-cli) added a built-in plugin system to kubectl that allows anyone to add new sub-commands. This does not require editing kubectl’s source code or recompiling it.

Because oc is built on top of kubectl it benefits from the same plugin system.  This allows us to use the larger Kubernetes community of plugins available with our OpenShift clusters.

Any executable file in your PATH that starts with kubectl- or oc- can be called with the oc command. A plugin can be as simple as a bash scripts or a more complex applications written in a compiled language like Golang.

Let's look at creating a simple HelloWorld plugin to see how plugins work.
https://asciinema.org/a/380076

Discovering, installing, updating plugins manually can be a tedious task.   Enter [Krew](https://krew.sigs.k8s.io/) was devolped by 
#### Getting started with Krew

#### Useful plugins

> neat
> status
> get-all
> pod events/sick-pods
> rbac-lookup
> ksniff

#### Conclusion