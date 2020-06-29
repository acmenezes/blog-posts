# Linux Capabilities in OpenShift

## What are Linux Capabilities?

We know that Linux has two types of users. Privileged and Unprivileged. Privileged processes will bypass certain kernel permission checks and will be capable of doing almost everything in the system. Unprivileged processes are subjected to kernel permission checks which means the process credentials will be verified. So permissions granted for the user ID, group ID and/or supplementary group ID will tell what that specific process can do.

Some tasks are only allowed to be performed by the root user, user ID zero or superuser. In order to allow an unprivileged process to perform those tasks certain flags were created. Those flags are used by the kernel to check the process permissions for a specific task. These are what we call capabilities. In other words capabilities grant granular permissions on specific "privileged" tasks to unprivileged processes.

Taking from man pages we get this definition: "Starting  with kernel 2.2, Linux divides the privileges traditionally associated with superuser into distinct units, known as capabilities, which can be independently enabled and disabled." Check [man(7) capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html).

A very repeated example in books and blogs is the one where we need our application to bind a socket to a privileged port. Technically speaking, they are called the "well known port numbers", ranging from 0 to 1024, and were defined in [RFC 1340](https://tools.ietf.org/html/rfc1340#page-9). One of the most important pillars of internet's networking. And they can be used for both TCP and UDP protocols.

The story here is simple. We have an application that we want to serve on port 80 or 443. But we don't want that process to be privileged. Capability seems to be an amazing solution for that. We then use CAP_NET_BIND_SERVICE and nothing else. That should be enough to bind our service to a port running an application process under an unprivileged user.

  **Example Here**

So, at the moment we learn what capabilities are, the idea of having a white list of specific tasks as a security mechanism narrowing down the permissions we would like to give to our applications immediately pops up in our minds. But this subject is not that simple as it seems in the containerized world. When we talk about container platforms, container engines and container runtimes we have a very complex and not very clear path on how to configure capabilities for containers. And worse than that, at the time of this writing, some of the most recent features in the Linux kernel are not yet addressed by those platforms.

In the case of the example above one of the most intriguing questions is why can't I bind a server or service with non-root user to a socket on a privileged low number port inside a container running on Kubernetes or OpenShift if the container has its own network namespace? Even with CAP_NET_BIND_SERVICE!

 First of all the network stack is a clone of the root namespace network stack and is compliant to all standards and RFCs such as the one cited above. And that is simply because the protocols are the same and use the same code. It leads us to think smart and ask the same question but with reverse logic. Why can't I bind to a non privileged port then? Is there really any good reason not to do so?

 But let's say we have a use case for this and maybe other use cases that will pass through the same problem that is analyzing the required capabilities to have a "semi-privileged" environment (or processes + binaries + platform). Then we need to go deeply into weeds of how the Linux kernel treats those privileges.

#### Process Capabilities and File Capabilities

One important note before diving into this subject is that both processes and files can have capability sets. The process capabilities are tied to its user or inherited from its parent process. And here I may refer to process kind of interchangeably with threads or tasks. If needed I'll specify whether it's a parent process or a child one. And when we refer to file capabilities we talk about another level of permission that is encoded in the binary file extended attributes being ran by that process. Which means that if the process has proper capabilities but the binary that the process is trying to run doesn't it may be denied due to the absence of a capability of some sort. That we'll explore a bit further. But it's important to know that both the process and the binary it runs must be checked when troubleshooting capabilities.

If you want to take a look at the history behind file capabilities check this article [here](https://lwn.net/Articles/256519/).

## Process Capability Sets

Things begin to get more complicated when we look closer to the Linux code base. How are Linux Capabilities implemented? At a high level overview we used the word flags. Ok. But what flags? So capabilities are stored in a data structure that can be associated with processes and files. We've said that already. 

And when we talk about processes the information to manage and limit the action space for those specific privileged tasks is organized into 5 different sets. Below is a summary of the documentation that you can check in full [here](https://man7.org/linux/man-pages/man7/capabilities.7.html).

`Permitted Capabilities`

Is the superset that limits what can be in Effective and Inherited capabilities set.

`Inherited Capabilities`

Is the set of capabilities to preserved when a program calls [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html) to run another one that would inherit those capabilities.

> :warning: What is important here is that if the caller program is not running under a privileged user those capabilities won't be preserved and then ambient capabilities must be used in that case.

`Effective Capabilities`

"This is the set of capabilities used by the kernel to perform permission checks for the thread."

`Bounding Capabilities`

This is another layer of capability limitation applied to the thread when it tries to raise a new capability.

`Ambient Capabilities`

"This is a set of capabilities that are preserved across an [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html) of a program that is not privileged."

The problem it's trying to solve is described in details [here](https://lwn.net/Articles/636533/).

> :warning: A few comments about containers and orchestration platforms. 
>
>As of this point in time CRIO doesn't have the ability to handle ambient capabilities as we see in the portion of code below:

```
func setupCapabilities(specgen *generate.Generator, capabilities *pb.Capability) error {
	// Remove all ambient capabilities. Kubernetes is not yet ambient capabilities aware
	// and pods expect that switching to a non-root user results in the capabilities being
	// dropped. This should be revisited in the future.
	specgen.Config.Process.Capabilities.Ambient = []string{}

```
> That source code can be found [here](https://github.com/cri-o/cri-o/blob/d0dc0d3076367f6a26c46fb6e72d1e582b552599/server/container_create.go#L375-L379) if it's not changed by the time you read it.
>
> And the discussion around ambient capabilities can be found [here](https://github.com/kubernetes/kubernetes/issues/56374). It's an issue opened in November 25 of 2017 and it remains under discussion by June 2020.


## File Capability Sets

And when we talk about file capabilities they are stored in the file extended attributes. And we have:

"The file capability sets, in conjunction with the capability sets of the thread, determine the capabilities of a thread after an [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html)."

`Permitted Capabilities`

If set in the file they are automatically permitted to the thread running it.

`Inherited Capabilities`

It sums up with the thread's inherited capability set to get the final one.

`Effective Capabilities`

And finally, on files effective capabilities are just a bit that enables all permitted capabilities to be added to the final effective capability set of the thread.

> In other words file capabilities extend the thread's capabilities.

## Capability-aware programs:

There are two types of programs in this case. The ones in binary files that were marked in their attributes as having capabilities to be addedd to the thread with the effective bit set to true and the ones that actually recognize and understand capabilities because they are running system calls in their code defined in [libcap](https://man7.org/linux/man-pages/man3/libcap.3.html) in order to manipulate capability information and configuration.

On the [capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html) man page under the section "Safety checking for capability-dumb binaries" it becomes clear what it means.


**Example with All Capabilities Included - The setuid bit Set**

Testing in a Ubuntu Xenial distro with the ping command we can see that it doesn't have any capabilities associated with it.

`getcap /bin/ping` doesn't output anything.

But when looking at the ls -l /bin/ping we can see:

```
-rwsr-xr-x 1 root root 64424 Jun 28  2019 /bin/ping
```

Look at the `s` in the permission field. It means that `setuid`is set to true and that the execution of the file will happen with, in this case, the owner's permissions which happens to be root. Therefore all capabilities would be included here to send a simple ICMP message over the network.

**Example with an Specific Capability Set**

Let's try the same in a CentOS 7 distro.
```
$ ls -l /usr/bin/ping
-rwxr-xr-x. 1 root root 66176 Aug  4  2017 /usr/bin/ping
```
First of all we see no `s` in the permission field. Now let's get caps.
```
$ getcap /usr/bin/ping
/usr/bin/ping = cap_net_admin,cap_net_raw+p
```
Here the permitted capability set has the caps above and the effective set has the bit flipped to true which adds those 2 permitted capabilities to the permitted capability set and effective capability set belonging to the process running it that may be under a regular user. That's how it get permission to execute the tasks. But it's important to say that as long as you run whatever binary it is it will have those capability raised during all the lifetime of that thread because they are configured in the extended attributes on the file system. Even if the task it needed to be performed runs only once at during the process lifetime.

**Example Cap Aware**






## Permission Check with Capabilities Step by Step

Here I present an attempt to make sense of what we see in the Linux documentation, specially in the [capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html). It's not a definitive guide since there are other variables that may need to be analyzed that are not taken into consideration for the sake of being simpler and/or the lack of knowledge at this point. But still, I think it worth the effort to have a visual representation on that. So here it is a diagram on how the algorithm would be treating permission checks when capabilities are into play:

**Diagram Here**

> Please, if there is any kernel developer out there that want to contribute with this I'll be glad to hear you. Just contact me at amenezes@redhat.com

## Verifying and Tracing Capabilities:

As we saw in the previous sections, binaries can be capability aware. That means that they can understand and manipulate capabilities on the fly. So new capabilities can be raised, lowered or dropped.

That's one of the reasons why the tool `capsh` may not be the best way to analyse capabilities since it "greps" the actual state of the thread's capability sets. Of course it can be used to test binaries ad hoc and let us understand a bit how our application is depending on capabilities to accomplish some tasks.

**capsh with normal user**

```
capsh --print -- -c "ping 127.0.0.1"
Current: =
Bounding set =cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,35,36
Securebits: 00/0x0/1'b0
 secure-noroot: no (unlocked)
 secure-no-suid-fixup: no (unlocked)
 secure-keep-caps: no (unlocked)
uid=1000(vagrant)
gid=1000(vagrant)
groups=993(docker),1000(vagrant)
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.108 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.037 ms
64 bytes from 127.0.0.1: icmp_seq=3 ttl=64 time=0.089 ms
```

**capsh with `sudo`**
```
sudo capsh --print -- -c "ping 127.0.0.1"
Current: = cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,35,36+ep
Bounding set =cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,35,36
Securebits: 00/0x0/1'b0
 secure-noroot: no (unlocked)
 secure-no-suid-fixup: no (unlocked)
 secure-keep-caps: no (unlocked)
uid=0(root)
gid=0(root)
groups=0(root)
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.042 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.086 ms
64 bytes from 127.0.0.1: icmp_seq=3 ttl=64 time=0.083 ms
```

**How to get the other sets**

Let's put a ping to loopback in the background:

```
ping 127.0.0.1 > /dev/null &
[1] 718
```

Now let's grep its status:
```
grep Cap /proc/718/status
CapInh: 0000000000000000
CapPrm: 0000000000003000
CapEff: 0000000000000000
CapBnd: 0000001fffffffff
CapAmb: 0000000000000000
```

Now let's use `capsh --decode=` to decode it:
```
for line in $(grep Cap /proc/718/status | awk '{print $2}'); do capsh --decode=$line; done;
0x0000000000000000=
0x0000000000003000=cap_net_admin,cap_net_raw
0x0000000000000000=
0x0000001fffffffff=cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,35,36
0x0000000000000000=
```

**Example capsh adding capabilties**




**Example capsh removing capabilities**




**Example Using `getcap` to analyze file capabilities:**
```
$ which ping
/usr/bin/ping
$ getcap /usr/bin/ping
/usr/bin/ping = cap_net_admin,cap_net_raw+p
```
> :warning: Please remark that this ping binary comes from a CentOS machine. In other Linux distros you may find that there is no capbility bit set at all and the ping command is actually running with root privileges with the setuid bit turned to true. But that's another story.

**Using iovisor to trace capabilities**

For those that don't know the iovisor project I definitely recommend taking a look [here](https://www.iovisor.org) and [here](https://github.com/iovisor). I'm particularly interested for the sake of this article in the [BCC tools](https://github.com/iovisor/bcc). Specially the one developed by Brendan Gregg that you can find at ./tools/capable.py.

We already talked about dynamic capability manipulation. This is the tool that can grab some information form kernel in poll mode like regarding process capabilities on the fly. Some explanation about that can be found [here](http://www.brendangregg.com/blog/2016-10-01/linux-bcc-security-capabilities.html) and also [here](https://stackoverflow.com/questions/35469038/how-to-find-out-what-linux-capabilities-a-process-requires-to-work).

#### libcap-ng

  - pscap, netcap and filecap

## Capabilities and Containers

Let's take a few steps back and try to picture what we have here. Capabilities are assigned to processes and files. Containers can be defined as isolated processes with their own file systems into namespaces and cgroups. Runtimes create or, better, automate the creation of those isolated processes. Container engines are the piece that talks to the actual runtime passing the configuration parameters for container creation. Orchestration platforms have the abstraction layer that will use those engines to spin all their pods and deployments.

Let's recap here. The flow of configuration could be simplified as:

Pod.yaml --> Kubernetes --> CRIO --> Runc --> Linux Container

**Api Call Diagram Here**

First and probably most important information here is: where goes the capability information in each phase of configuration in order to get to the container itself?

Basically:

**Diagram here**

SecurityContext (Container Spec) + SCCs -->  Default Capabilities List (crio.conf) -->  5 different process cap sets (runc conf.json)


So something is missing in those types, right? There is no field for the 5 different capability sets in the Pod/Container Spec type or in the CRIO configuration file. Here is where things begin to get muddy. If for some reason there is a need for ambient capabilities, which is exactly the case when we want to use non-root users, those capabilities will be dropped in the end of the line for reasons we explored in previous sections. That said if the binary file is not capability aware or has it's own permitted set configured with the effective bit set to true it will get a permission denied error.

## Demo on OpenShift

If you want to try for yourself check the example below.

**All demo here or link to it**

## Some Possible use Cases and Approaches

#### Networking

    Multiple network vendors and telecom users

    - port binding
    - network configuration
    - pinging
    - The Sysctl approach

#### Storage

  Architecture considerations - unix sockets vs networked communication

    - nsm case  - Mount, read and write from/to privileged paths
    - unix sockets

#### Memory lock use cases

    - IPC_LOCK capability 

    vault case

    vpp case

#### Some General Security Guide Lines

  - Something with super user powers should be trusted not constrained LXC position

  - Capabilities within non root namespaces are more secure


## Conclusion


#### References

Books

Chapter 8. Linux Kernel Security, Capabilities, and Seccomp in "Observability with BPF"
by David Calavera; Lorenzo Fontana
Published by O'Reilly Media, Inc., 2019

Chapter 11. Security in "BPF Performance Tools: Linux System and Application Observability"
by Brendan Gregg
Published by Addison-Wesley Professional, 2019

Chapter 8. Running Containers - Container Security in "Cloud Native DevOps with Kubernetes"
by John Arundel; Justin Domingus
Published by O'Reilly Media, Inc., 2019

Articles and Documentation:

capabilities(7) from Man Pages
https://man7.org/linux/man-pages/man7/capabilities.7.html

getcap, setcap and file capabilities, 2017
https://www.insecure.ws/linux/getcap_setcap.html

Ambient Capabilities 
by Andy Lutomirski, 2015
https://lwn.net/Articles/636533/

Linux capabilities support for user namespaces
by Jake Edge, 2010
https://lwn.net/Articles/420624/

Fixing CAP_SETPCAP
by Jake Edge, 2007
https://lwn.net/Articles/256519/

Linux bcc Tracing Security Capabilities
by Brendan Gregg, 2016
http://www.brendangregg.com/blog/2016-10-01/linux-bcc-security-capabilities.html

Linux Capabilities: Why They Exist and How They Work
By Adrian Mouat
https://blog.container-solutions.com/linux-capabilities-why-they-exist-and-how-they-work

Linux Capabilities In Practice
By Adrian Mouat
https://blog.container-solutions.com/linux-capabilities-in-practice

Videos

Capable - Beginning at 00:10:00 - https://www.youtube.com/watch?v=44nV6Mj11uw
BSidesSF 2017 - Linux Monitoring at Scale with eBPF (Brendan Gregg & Alex Maestretti)

Github Issues:

Kubernetes should configure the ambient capability set
https://github.com/kubernetes/kubernetes/issues/56374

Can't bind to privileged ports as non-root
https://github.com/moby/moby/issues/8460

Add support for ambient capabilities 
https://github.com/moby/moby/pull/26979