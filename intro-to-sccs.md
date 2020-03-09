# Introduction to Security Contexts and SCCs
Written by Alexandre Menezes

With Role Based Access Control we have an Openshift wide tool to determine what each user can do, meaning the verbs that can be used, against what objects in the API. For that, rules are defined combining resources with the API verbs into sets called roles and with the role binding we attribute those rules to users. Once we have those Users or Service Accounts, we can attribute them to particular resources to give them access to those actions. For example, a Pod may be able to delete a ConfigMap but not a Secret when running under a specific Service Account. That's an upper level control plane feature that doesn't take into account the underlay node permission model, meaning the unix permission model, and some of it's kernel newer sophistications. 

So, the container platform is protected with good RBAC practices from it's created objects but the node may be not. That is where a Pod may not be able to delete an object in etcd using the API because it's restricted by RBAC but it may delete important files in the system and even stop kubelet if properly programmed for that. There comes the SCCs or Security Context Constraints for the rescue.

### Linux Processes and Privileges

Before going into deep waters with SCCs, let's go back in time and take a look at some of the key concepts Linux brings to us regarding processes. A good start is entering the command [man capabilities](http://man7.org/linux/man-pages/man7/capabilities.7.html) on a Linux terminal. That's the manual page that contains very important fundamentals to understand the goal behind the SCCs.

The first important distinction that we need to do is between privileged and unprivileged processes. While privileged processes will have user ID 0 being the superuser or root, unprivileged processes will have non-zero user IDs. Privileged processes bypass kernel permission checks. That means that the actions that a process or thread can perform on operating systems objects such as files, directories, symbolic links, pseudo filesystems (procfs, cgroupfs, sysfs etc.) and even memory objects such as shared memory regions, pipes and sockets… Those actions are unlimited and not verified by the system. Meaning, the kernel won't check user, group or others permissions (taking from the Unix permission model UGO - user, group and others) to grant access to that specific object in behalf of the process.

If we look at the list of running processes on a Linux system using the command `ps -u root` we will find very important processes such as systemd for example that has the PID 1 and is responsible for bootstrapping the user space in most distributions and initializing most common services. For that it needs non restricted access to the system.

Unprivileged processes, though, are subject to full permission checking based on process credentials (user ID, group ID and supplementary group list etc.). The kernel will make an iterative check under each category user, groups and others trying to match the user and group credentials on the running process with the target object's permissions in order to grant or deny access. Keep in mind that this is not the service account in Openshift. This is the system's user that runs the container process if we want to speak containers.

After kernel 2.2 the concept of capabilities was introduced. In order to have more flexibility and  enable the use of superuser or root features in a granular way, those super privileges were broken into small pieces that can be enabled or disabled independently. That is what we call capabilities. We can take a deeper look on http://man7.org/linux/man-pages/man7/capabilities.7.html

As an example, let's say that we have an application that needs special networking configurations. Let's say that we need to configure one interface, open a port on the system's firewall, create a NAT rule for that and punt a new custom route on the system's routing table. But you don't need to make arbitrary changes to any file in the system. We can set CAP_NET_ADMIN instead of running the process as a privileged one.

Beyond privileges and capabilities we have SELinux and AppArmor that are both kernel security modules that can be added on top of capabilities to get even more fine grained security rules by using access control security policies or program profiles. In addition, we have Seccomp which is a secure computing mode kernel facility that reduces the available system calls to the kernel for a given process.

Finally, adding to all that, we still have interprocess communications, privilege escalation and access to the host namespace when we begin to talk about containers. That is out of scope here at this point but...

### How does that translate to containers?

That said, we come back to containers and ask: what are containers again? They are processes segregated by namespaces and cgroups and on that note they have all the same security features described above. So how do we create containers with those security features then?

Let's first take a look at what is the smallest piece of software that creates the container process: runc. As its definition on the github page says, it's a tool to spawn and run containers according to the [OCI specification](https://github.com/opencontainers/runtime-spec). It's the default choice for OCI runtimes although we have others such as kata containers. In order to use runc, we need to have a file system image and a bundle with the configuration for the process. The short story on the bundle is we must put a json formatted specification for the container where all the configurations will be taken into account. Check this part of it's documentation: https://github.com/opencontainers/runtime-spec/blob/master/config.md#linux-process

From there we have fields such as apparmorProfile, capabilities or selinuxLabel. We can set user ID, group ID and supplementary group IDs. What tool then automates the process of getting the file system ready and passing down those parameters for us?

We can use podman, for example, for testing or development, running isolated containers or pods. It allows us to do it with special privileges as we show below:

Privileged bash terminal:
`sudo podman run --privileged -it registry.access.redhat.com/rhel7/rhel /bin/bash`

Process ntpd with privilege to change the system clock:
`sudo podman run -d --cap-add SYS_TIME ntpd`


Ok. Cool. But when it comes the time to run those containers on Kubernetes or Openshift how do we configure those capabilities and security features? 

Inside the Openshift platform CRI-O container engine is the one that runs and manages containers. It is compliant with the Kubernetes Container Runtime Interface (CRI). It complies with kubelet rules in order to give it a standard interface to call the container engine and all the magic is done automating runc behind the scenes while allowing other features to be developed under the engine itself.

Following the workflow above to run a pod in Kubernetes or Openshift, we'll first make an API call to kubernetes asking to run a particular Pod. It could come from an oc command or from code, for example. Then the API will process that request and store it in etcd; the pod will be scheduled for a specific node since the scheduler watches those events; finally, kubelet, in that node, will read that event and call the container runtime (CRI-O) with all the parameters and options requested to run the pod. I know it's very summarized. But the important thing here is that we need to pass parameters down to the API in order to have our Pod with the desired privileges configured. In the example below a new pod gets scheduled to run in node 1.

<img src='img/Openshift API Call.png'></img>

What goes into that yaml file in order to request those privileges? Two different objects are implemented under the Kubernetes API: PodSecurityContext and SecurityContext. The first one, obviously, related to Pods and the second one related to the specific container. They are part of their respective types. So you can find those fields on Pod and Container Specs on yaml manifests. With that they can be applied to an entire Pod, no matter how many containers are there or to specific containers into that Pod. Then the SecurityContext settings take precedence over the PodSecurityContext ones. You can find the security context source code under https://github.com/kubernetes/api/blob/master/core/v1/types.go.

[Here](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) we can find a few examples on how to configure security contexts for Pods. Below I present the first three fields of the SecurityContext object.

```
type SecurityContext struct {
	// The capabilities to add/drop when running containers.
	// Defaults to the default set of capabilities granted by the container runtime.
	// +optional
	Capabilities *Capabilities `json:"capabilities,omitempty" protobuf:"bytes,1,opt,name=capabilities"`
	// Run container in privileged mode.
	// Processes in privileged containers are essentially equivalent to root on the host.
	// Defaults to false.
	// +optional
	Privileged *bool `json:"privileged,omitempty" protobuf:"varint,2,opt,name=privileged"`
	// The SELinux context to be applied to the container.
	// If unspecified, the container runtime will allocate a random SELinux context for each
	// container.  May also be set in PodSecurityContext.  If set in both SecurityContext and
	// PodSecurityContext, the value specified in SecurityContext takes precedence.
	// +optional
	SELinuxOptions *SELinuxOptions `json:"seLinuxOptions,omitempty" protobuf:"bytes,3,opt,name=seLinuxOptions"`
    <...>
}
```

Here is an example of a yaml manifest configuration with capabilities on securityContext field:

```
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-4
spec:
  containers:
  - name: sec-ctx-4
    image: gcr.io/google-samples/node-hello:1.0
    securityContext:
      capabilities:
        add: ["NET_ADMIN", "SYS_TIME"]
```
Ok. Now what? We have an idea on how to give super powers to a container or Pod even though they may be RBAC restricted. How can we control this behavior?


### Security Context Constraints

Finally we get back to our main subject. How can I make sure that a specific Pod or Container doesn't request more than what it should request in terms of process privileges and not only Openshift object privileges under it's API? 

That's the role of Security Context Constraints. To check beforehand if the system can pass that pod or container configuration request, with privileged or custom security context, further onto the cluster API that will end up running a powerful container process. To have a taste on what a SCC looks like here is an example:

```
oc get scc restricted -o yaml

allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: true
allowPrivilegedContainer: false
allowedCapabilities: null
apiVersion: security.openshift.io/v1
defaultAddCapabilities: null
fsGroup:
  type: MustRunAs
groups:
- system:authenticated
kind: SecurityContextConstraints
metadata:
  annotations:
    kubernetes.io/description: restricted denies access to all host features and requires
      pods to be run with a UID, and SELinux context that are allocated to the namespace.  This
      is the most restrictive SCC and it is used by default for authenticated users.
  creationTimestamp: "2020-02-08T17:25:39Z"
  generation: 1
  name: restricted
  resourceVersion: "8237"
  selfLink: /apis/security.openshift.io/v1/securitycontextconstraints/restricted
  uid: 190ef798-af35-40b9-a980-0d369369a385
priority: null
readOnlyRootFilesystem: false
requiredDropCapabilities:
- KILL
- MKNOD
- SETUID
- SETGID
runAsUser:
  type: MustRunAsRange
seLinuxContext:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
users: []
volumes:
- configMap
- downwardAPI
- emptyDir
- persistentVolumeClaim
- projected
- secret
```
That above is the default SCC that has pretty basic permissions and will accept Pod configurations that don't request special security contexts. Just by looking at the name of the fields we can have an idea on how many features it can verify before letting a workload with containers pass by the API and get scheduled.

In conclusion, we have at hand a tool that allows an Openshift admin to decide whether an entire pod can run in privileged mode, have special capabilities, access directories and volumes on the host namespace, use special SELinux contexts, what ID the container process can use among other features before the Pod gets requested to the API and passed to the container runtime process. 

In the next blog posts we'll explore each field of an SCC, explore their underlying Linux technology, present the prebuilt ones and understand their relationship with the RBAC system to grant or deny special security contexts declared under Pod's or container's Spec field. Stay tuned!
