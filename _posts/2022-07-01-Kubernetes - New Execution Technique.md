---
title: Kubernetes - New Execution Technique
author: Khush V
date: 2022-07-10 00:00:00 +0000
categories: [AWS, Security Specialty Certification]
tags: [AWS, cloud, certification]
---

# Kubernetes - Red Team Musings

Having recently conducted several red team exercises, where the objectives involved compromising Kubernetes clusters, I had the opportunity to explore Kubernetes from a red team perspective. I would like to present a new technique  mapped to Execution stage to the Threat  Matrix for Kubernetes developed by Microsoft[^1]. This Threat Matrix is an attempt to methodically map different tactics, techniques and procedures against Kubernetes clusters against the different stages of a threat actor's lifecycle.

![Threat Matrix - Microsoft](/assets/img/kubernetes_1/threat_matrix.png)
_Threat Matrix by Microsoft._

The techniques for gaining execution on a Kubernetes cluster are as follows:
- Exec into container
- bash/cmd inside container
- New container
- Application Exploit (RCE)
- SSH server running inside container 
- Sidecar injection

The new method involves lifecycle hook, which does not spawn any new non-application specific containers. In this approach, this technique should fly under the radar in the instances where the number of containers running is monitored closely.

This guide assumes a general knowledge of Kubernetes. For a better understanding of clusters and how they work, I recommend the official documentation[^2], which explains the foundations very well.

## Lifecycle Hook

Pods have a lifecycle during their time in a Kubernetes cluster. From a high level view, when a pod is deployed, it's status begins as `Pending`. If at least one of the containers within the pod is able to be started and executed, the status of the pod changes to `Running`. Upon termination, the status can change to `Succeeded` or `Failed` depending the exit code of the process within the container. 

Kubernetes has a feature that allows the user to hook a handler to a lifecycle event. This allows a handler to execute code when the state of a pod changes. When a pod changes state, Kubernetes will execute the command attached to the appropriate lifecycle handler. The lifecycle hook executes the command as if within the container, and shares the underlying file system access. A beacon, if executed with a lifecycle handler  will then be able to access the fileand folders in the container. 

There are two types of lifecycle events we can hook: `PostStart` and `PreStop`. `PostStart` events that triggered the moment a container is created. Note that there is no guarantee that the handler will execute before the container `ENTRYPOINT` is reached. `PreStop` events are generated just before a container is due to be terminated by the scheduler. If the `preStop` handler function exceeds the `terminationGracePeriodSeconds` value in seconds, it will fail.  

Let's see an example. Let's create pod, with a lifecycle handler defined that will hook a PostStart event. The lifecycle handler will execute a command when the container is started:

```
apiVersion: v1
kind: Pod
metadata:
  name: post-start-hook-demo
spec:
  containers:
  - name: demo
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /tmp/message"]

```
The handler is declared using the `lifecycle` directive and we can specify the command we wish to execute when the `postStart` event is triggered. We deploy this into a Kubernetes cluster:

![Deploying a pod with postStart handler](/assets/img/kubernetes_1/create-pod.png)
_Deploying a pod with postStart handler._

If we exec into the container, we can see the file created in `/tmp` and the contents written to the file as per the manifest above:

![Viewing the output of the postStart handler](/assets/img/kubernetes_1/pod-exec.png)
_Viewing the output of the postStart handler._


## Indicator of Attack

For defenders, If we `describe` the pod, it is possible to an Indicator of Attack within the events table. The output and status of the lifecycle handlers can be seen here. 

![Indicator of Attack](/assets/img/kubernetes_1/pod-describe.png)
_Indicator of Attack._



## Conclusion

For me personally, exec'ing into containers and spawning new or sidecar containers has worked very well. Although lately I've noticed a trend in preventing users from dropping into a container, especially in production environments. In these cases, hooking a lifecycle event is perfect when you don't want to spawn a new container or sidecar container.

## Links
[^1]: [Threat matrix for Kubernetes](https://www.microsoft.com/security/blog/2021/03/23/secure-containerized-environments-with-updated-threat-matrix-for-kubernetes/)
[^2]: [https://kubernetes.io/docs/home/](https://kubernetes.io/docs/home)
[^3]: [AWS Official Course](https://www.aws.training/Details/eLearning?id=34786)
[^4]: [Kubernetes Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)
