



Workload Admission Flow in OpenShift


Processes and Permissions


Pod Security Admission

<!-- # The admission validates only
# There are only three predefined security levels
# The admission works on a per-namespace basis  -->

- Pod Security By Namespace - Pod Security Standards

- OpenShift defaults and custom configuration

<!-- # Based on the above, the user is now fully responsible to configure their pods’ 
# securityContext in order to be able to match a given pod security standards profile.  -->

Security Context Constraints Version 2

<!-- V2 does not permit allowPrivilegeEscalation=true 
Empty or false is compatible with v1 SCC and therefore works on OCP versions < 4.11
V2 requires you to leave the dropped capabilities empty, set it to ALL, or add only NET_BIND_SERVICE
By being accepted as v2 the SCC will always drop ALL. V1 only dropped KILL, MKNOD, SETUID, SETGID capabilities.
V2 still allows explicitly adding the NET_BIND_SERVICE capability
V2 requires you to either leave SeccompProfile empty or set it to runtime/default
Empty is compatible with v1 and works on OCP versions < 4.11 -->

- Privilege escalation false and what that means for File Capabilities

Linux Capabilities and Changes in CRIO


How to verify Pods and Container processes permissions


