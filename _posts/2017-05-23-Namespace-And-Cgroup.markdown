---
layout: post
title:  "Namespace AND Cgroup"
date:   2017-05-23 17:25:00 +0800
categories: container
---

## NAMESPACE
### [Namespaces in operation, part 1](https://lwn.net/Articles/531114/)  

[chroot jail](https://unix.stackexchange.com/questions/105/chroot-jail-what-is-it-and-how-do-i-use-it)

> A chroot jail is a way to isolate a process and its children from the rest of the system. It should only be used for processes that don't run as root, as root users can break out of the jail very easily.
> 
> The idea is that you create a directory tree where you copy or link in all the system files needed for a process to run. You then use the chroot() system call to change the root directory to be at the base of this new tree and start the process running in that chroot'd environment. Since it can't actually reference paths outside the modified root, it can't perform operations (read/write etc.) maliciously on those locations.
> 
> On Linux, using a bind mounts is a great way to populate the chroot tree. Using that, you can pull in folders like /lib and /usr/lib while not pulling in /user, for example. Just bind the directory trees you want to directories you create in the jail directory.

`chroot jail`实际上就是调用了`chroot`的进程无法操作除了指定目录以外的文件路径，就像一个`jail`；实际中，可以使用`bind mount`来创建`chroot jail`  

总共有六种`namespace`:  

> 1. `MOUNT NAMESPACE`, `CLONE_NEWNS`
> 2. `IPC NAMESPACE`, `CLONE_NEWIPC`
> 3. `PID NAMESPACE`, `CLONE_NEWPID`
> 4. `UTS NAMESPACE`, `CLONE_NEWUTS`
> 5. `USER NAMESPACE`, `CLONE_NEWUSER`
> 6. `NETWORK NAMESPACE`, `CLONE_NEWNET`

`UTS namespaces` (CLONE_NEWUTS, Linux 2.6.19) isolate two system identifiers—`nodename` and `domainname`—returned by the `uname()` system call; the names are set using the `sethostname()` and `setdomainname()` system calls. In the context of containers, the UTS namespaces feature allows each container to have its own hostname and NIS domain name. This can be useful for initialization and configuration scripts that tailor their actions based on these names. The term "UTS" derives from the name of the structure passed to the uname() system call: struct utsname. The name of that structure in turn derives from "**UNIX Time-sharing System**".

### [Namespaces in operation, part 2: the namespaces API](https://lwn.net/Articles/531381/)

#### Creating a child in a new namespace: `clone()`  

两个没见过的宏: `EXIT_SUCCESS`, `EXIT_FAILUER`  

Refer to [cppreference](http://en.cppreference.com/w/c/program/EXIT_status)

> The `EXIT_SUCCESS` and `EXIT_FAILURE` macros expand into integral expressions that can be used as arguments to the exit function (and, therefore, as the values to return from the main function), and indicate program execution status.

Constant | Explanation 
---|---
| EXIT_SUCCESS | successful execution of a program | 
| EXIT_FAILURE | unsuccessful execution of a program |

##### `/proc/$$/ns`  

The `/proc/PID/ns` symbolic links also serve other purposes. If we open one of these files, then the namespace will continue to exist as long as the file descriptor remains open, even if all processes in the namespace terminate. The same effect can also be obtained by **bind mounting one of the symbolic links to another location in the file system**:

``` bash
# touch ~/uts # Create mount point
# mount --bind /proc/27514/ns/uts ~/uts
```

### [Namespaces in operation, part 3: PID namespaces](https://lwn.net/Articles/531419/)  

### [Mount namespaces and shared subtrees](https://lwn.net/Articles/689856/)  

When the system is first booted, there is a single mount namespace, the so-called "**initial namespace**". New mount namespaces are created by using the CLONE_NEWNS flag with either the clone() system call (to create a new child process in the new namespace) or the unshare() system call (to move the caller into the new namespace). When a new mount namespace is created, it receives **a copy of the mount point list replicated from the namespace of the caller of clone() or unshare()**.

### [Namespaces in operation, part 5: User namespaces](https://lwn.net/Articles/532593/)  

The second point of interest is the user and group IDs of the child process. **As noted above, a process's user and group IDs inside and outside a user namespace can be different**. However, **there needs to be a mapping from the user IDs inside a user namespace to a corresponding set of user IDs outside the namespace**; the same is true of group IDs. This allows the system to perform the appropriate permission checks when a process in a user namespace performs operations that affect the wider system (e.g., sending a signal to a process outside the namespace or accessing a file).

System calls that return process user and group IDs—for example, getuid() and getgid()—always return credentials as they appear inside the user namespace in which the calling process resides. **If a user ID has no mapping inside the namespace, then system calls that return user IDs return the value defined in the file /proc/sys/kernel/overflowuid, which on a standard system defaults to the value 65534**. Initially, a user namespace has no user ID mapping, so all user IDs inside the namespace map to this value. Likewise, a new user namespace has no mappings for group IDs, and all unmapped group IDs map to /proc/sys/kernel/overflowgid (which has the same default as overflowuid).  

### Mapping user and group IDs

Normally, one of the first steps after creating a new user namespace is to define the mappings used for the user and group IDs of the processes that will be created in that namespace. This is done by writing mapping information to the `/proc/PID/uid_map` and `/proc/PID/gid_map` files corresponding to one of the processes in the user namespace. (Initially, these two files are empty.) This information consists of one or more lines, each of which contains three values separated by white space:

Defining a mapping is a **one-time** operation per namespace: we can perform only a single write (that may contain multiple newline-delimited records) to a uid_map file of exactly one of the processes in the user namespace. Furthermore, the number of lines that may be written to the file is currently limited to five (an arbitrary limit that may be increased in the future).

### Network namespace

```bash
# 添加 namespace
ip netns add ns1
ip netns add ns2
```

## [control group](https://www.kernel.org/doc/Documentation/cgroup-v2.txt)

`core`: 负责组织process hierarchy  
`controlle`: 负责分配资源  

/sys/fs/cgroup/

### [cgroup1](https://www.kernel.org/doc/Documentation/cgroup-v1/)

Each cgroup is represented by a directory in the cgroup file system
containing the following files describing that cgroup:

 - tasks: list of tasks (by PID) attached to that cgroup.  This list
   is not guaranteed to be sorted.  Writing a thread ID into this file
   moves the thread into this cgroup.
 - **cgroup.procs**: list of thread group IDs in the cgroup.  This list is
   not guaranteed to be sorted or free of duplicate TGIDs, and userspace
   should sort/uniquify the list if this property is required.
   Writing a thread group ID into this file moves all threads in that
   group into this cgroup.
 - **notify_on_release** flag: run the release agent on exit?
 - **release_agent**: the path to use for release notifications (this file
   exists in the top cgroup only)

show the cgroup for a PID:

```bash
cat /proc/PID/cgroup
```

#### cgroup subsystems

1. cpuset
2. memory
