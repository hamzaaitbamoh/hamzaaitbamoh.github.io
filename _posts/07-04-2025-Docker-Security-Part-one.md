# blog1

Status: Not started

# Container Hardening – Part 1: How to secure docker containers?

Docker is often imagined as a magic box, something isolated, safe, and self-contained. But in reality, Docker is just a thin abstraction over Linux features. If you want to secure your containers, you need to first understand how they actually work.

## Docker is Just a Process on the Host

![image.png](image.png)

When you start a container, you're not launching a VM. You're just running a regular Linux process one that shares the same kernel as your host system.

![image.png](766376a4-448c-4ec9-8a6b-3fe6cc018436.png)

![image.png](a3b29020-bab5-4d3d-8e7e-338cf21231eb.png)

You’ll see the container is a process running under Docker's supervision. It’s isolated using namespaces and cgroups, but it’s not truly separate from the host.

Inside the container, PID 1 is just the shell process 

But on the host, this same container appears as a regular Linux process 127800.

You’ll also see a couple of additional Docker-related processes around it; that’s just Docker doing its orchestration behind the scenes. What matters is: your container is simply a process running under the hood.

**-pid=host: Containers Can See the Host**
By default, Docker containers have their own PID namespace. But if you run:

![image.png](image%201.png)

Now your container shares the host's PID namespace. That means:

- It can see every process on the host
- It can send signals to them
- And depending on permissions, it may even read memory or interfere with other containers

This completely breaks process isolation.

## Privileged vs Unprivileged Containers

**Unprivileged (Default)**
Containers run with limited capabilities. They can't access most kernel modules, hardware devices, or perform sensitive operations like mounting arbitrary filesystems.

![image.png](image%202.png)

**Privileged**

![image.png](image%203.png)

Now your container has full access to host devices and all capabilities. It can:

- Mount file systems
- Access hardware devices
- Modify kernel parameters

In short: it’s basically root on the host. This is extremely dangerous in production.

## Linux Capabilities: Fine-Grained Privileges

Linux splits root privileges into smaller units called capabilities.

By default, Docker gives containers a set of capabilities like:

- **CAP_CHOWN**
- **CAP_NET_BIND_SERVICE**
- **CAP_KILL**

You can inspect them with: capsh --print

![image.png](image%204.png)

Imagine you’re running a containerized application that listens on port 80, like a basic web server.
To do this, your container needs only one capability: NET_BIND_SERVICE, which allows binding to ports below 1024 (privileged port).

By default, Docker grants many more capabilities than needed. To lock it down:
This command:

```bash
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx
```

- Drops all Linux capabilities (--cap-drop=ALL)
- Adds back only what's needed to bind to port 80 (--cap-add=NET_BIND_SERVICE)
- Runs the official nginx image, which by default listens on port 80

That’s it.. no need for full root powers, no file system control, no unnecessary privileges.

***Why This Is Important ?***
If that web server is ever compromised, the attacker won't be able to:

- Change ownership of files
- Mount or unmount file systems
- Load kernel modules
- Kill arbitrary processes

You’ve given it only what it needs, nothing more.

## Seccomp: Restricting Syscalls to What’s Necessary

Even with capabilities removed, a process inside a container can still use a wide range of Linux system calls. Some of these syscalls like ptrace, mount, clone, or reboot can be dangerous if abused.

That’s where seccomp comes in.

Seccomp (short for Secure Computing Mode) allows you to define a filter that restricts which syscalls a container is allowed to use. If a blocked syscall is used, the process is killed or denied access.

⇒ Default Protection
By default, Docker applies a built-in seccomp profile, which blocks around 44 risky syscalls (like keyctl, ptrace, mount, kexec_load, etc.).
So unless you've disabled it, Docker is already applying some basic syscall filtering.
Let’s return to our minimal NGINX example — now adding seccomp hardening:

```bash
docker run \
--cap-drop=ALL \
--cap-add=NET_BIND_SERVICE \
--security-opt seccomp=default \
nginx
This:
```

Drops all Linux capabilities

Adds back only the ability to bind to port 80

Applies Docker’s default seccomp profile, which blocks high-risk syscalls

**This means:**

Even if the app is vulnerable, it can’t call dangerous syscalls to escape, It can serve HTTP traffic but not mess with the kernel or host

For tighter control, you can define your own seccomp JSON profile and load it with:

```bash
docker run --security-opt seccomp=/path/to/profile.json myimage
```

But for most use cases, Docker’s default profile is a solid baseline.

## Namespaces: The Sandbox Foundation

Namespaces isolate six areas of the Linux system:

- **pid**: Process IDs
- **mnt**: Mount points (file systems)
- **net**: Network stack
- **ipc**: Interprocess communication
- **uts**: Hostname & domain
- **user**: User IDs

This is how Docker gives containers their own view of the world.
But if you share namespaces with the host (e.g. --pid=host, --net=host), you break the sandbox.

## AppArmor: Mandatory Access Control

Beyond namespaces and capabilities, Linux offers Mandatory Access Control (MAC) through: AppArmor which enforce strict policies:

- What files a process can access
- What system calls it can make
- What operations it can perform

Docker supports this via:

```bash
docker run --security-opt apparmor=docker-default alpine
```

## Docker Bench for Security

If you want to go even further in securing your Docker environment, consider running the [Docker Bench for Security](https://github.com/docker/docker-bench-security) script.

This tool checks your Docker configuration against the **CIS Docker Benchmark**, covering areas like:

**Host Configuration**
- OS and kernel hardening
- Audit configurations for Docker binaries

**Docker Daemon Configuration**
- TLS usage
- User namespace remapping
- Inter-container communication (`--icc`)
- Live restore feature

**Container Image & Runtime Checks**
- Usage of trusted and versioned images
- Avoiding `--privileged`
- Dropping unnecessary capabilities
- Applying seccomp, AppArmor, or SELinux profiles

**Security Operations**
- Proper Docker socket permissions
- Audit logging
- Secure credential storage

![image.png](image%205.png)

## Conclusion

Docker containers are just processes. They can be isolated, but only if you configure them correctly. Misusing options like --privileged or --pid=host can turn that isolation into a security illusion.