---
layout: post
title: "Kubernetes PKI and Security Architecture"
subtitle: "From Component Certificates to User Access and RBAC — explored through Minikube"
date: 2025-10-25 23:00:13 -0400
background: "/img/posts/cover.jpg"
---

# Understanding the Kubernetes Trust Model

Kubernetes is a system built on **trust and verification**.  
Every component — from the API server to your `kubectl` command — communicates through **mutual TLS (mTLS)**, proving its identity with a certificate signed by a **cluster Certificate Authority (CA)**.

Behind every command lies a sophisticated **Public Key Infrastructure (PKI)** that defines how Kubernetes knows:
- **Who** is talking (authentication)  
- **What** they’re allowed to do (authorization)  

In this deep dive, we’ll explore:
1. How Kubernetes components communicate securely using certificates.  
2. How the kubeconfig file encapsulates identity and trust.  
3. How to create and manage your own user certificate.  
4. How to use RBAC (Role-Based Access Control) to grant precise permissions.  

All examples use **Minikube**, but these principles apply to any Kubernetes cluster (kubeadm, AKS, EKS, GKE, etc.).

---

## 1. Certificates Between Kubernetes Components

### 1.1 Why Certificates Matter

The Kubernetes control plane is composed of several components : the API Server, Controller Manager, Scheduler, and others.. that must communicate securely. Every interaction between them uses TLS with certificates signed by the same CA. 

This ensures:
- **Integrity** — data can’t be altered in transit.  
- **Confidentiality** — all control-plane communication is encrypted.  
- **Authentication** — each component proves its identity before any exchange.  

![image.png](/img/posts/K8scerts/cia.png)

---

### 1.2 The Cluster PKI Layout

A Kubernetes control plane (deployed manually or via kubeadm) maintains its PKI under `/etc/kubernetes/pki/`.  
In Minikube, it’s located under `~/.minikube`.

| Component | Certificate | Key | Purpose |
|------------|-------------|-----|----------|
| **CA** | `ca.crt` | `ca.key` | Root of trust; signs all other certificates |
| **API Server** | `apiserver.crt` | `apiserver.key` | Serves HTTPS on port 6443 |
| **Controller Manager** | `controller-manager.crt` | `controller-manager.key` | Authenticates to the API server |
| **Scheduler** | `scheduler.crt` | `scheduler.key` | Authenticates to the API server |
| **Kubelet (Node)** | `kubelet.crt` | `kubelet.key` | Proves node identity |
| **Admin (kubectl)** | `admin.crt` | `admin.key` | Client certificate for the admin user |


![image.png](/img/posts/K8scerts/certs-components.png)

Hierarchy of trust:

![image.png](/img/posts/K8scerts/certs-components-sc.png)

### 1.3 Mutual Authentication Flow

1. **API Server** loads its certificate (`apiserver.crt`) and exposes the HTTPS endpoint.  
2. **Controller Manager**, **Scheduler**, and **Kubelets** connect using their own client certificates.  
3. The API Server validates their signatures against the CA.  
4. If valid, the connection is established; otherwise, it’s rejected.

This **mutual TLS (mTLS)** handshake ensures that every component is verified — no anonymous communication inside the cluster.

---

## 2. kubeconfig: The Bridge Between Users and the Cluster

### 2.1 What Is the kubeconfig File?

Your `~/.kube/config` is a YAML file that tells `kubectl`:
- **Which cluster** to talk to  
- **Which certificate** to use  
- **Which context** (user + namespace) is active

Example:

![image.png](/img/posts/K8scerts/kubeconfig.png)

### 2.2 Default Minikube Identity

Minikube automatically creates a client certificate with the following subject:

CN=minikube, O=system:masters

The “system:masters” group is bound by default to the ClusterRoleBinding named “cluster-admin”, giving full privileges to that user. This is why the default Minikube user can perform any action within the cluster.

![image.png](/img/posts/K8scerts/minikube-cert.png)

![image.png](/img/posts/K8scerts/sys.png)

## 3. Creating a Custom User and Certificate

To understand authentication beyond the built-in admin, we can create a new user manually.

![image.png](/img/posts/K8scerts/RBAC-2.png)

### Step 1 — Generate a Private Key

```bash
openssl genrsa -out hamza.key 2048
```
### Step 2 — Create a Certificate Signing Request

```bash
openssl req -new -key hamza.key -out hamza.csr -subj "/CN=hamza/O=dev-team"
```

Here, CN (Common Name) represents the username, and O (Organization) defines the group to which the user belongs.

### Step 3 — Sign the Certificate with the Cluster CA

```bash
openssl x509 -req -in hamza.csr
-CA ~/.minikube/ca.crt -CAkey ~/.minikube/ca.key
-CAcreateserial -out hamza.crt -days 365
```

This creates a valid client certificate trusted by the cluster’s CA.

![image.png](/img/posts/K8scerts/certs.png)

### Step 4 — Add the User to kubeconfig

Extend the kubeconfig file with the new user and context:

```yaml
users:

name: hamza
user:
client-certificate: /home/hamza/k8s-certs/hamza.crt
client-key: /home/hamza/k8s-certs/hamza.key

contexts:

name: hamza-context
context:
cluster: minikube
namespace: default
user: hamza
```

Then switch to that context:

```bash
kubectl config use-context hamza-context
```

When executing kubectl get pods, the API server authenticates the certificate but denies access because no authorization has been granted yet:

Error from server (Forbidden): User "hamza" cannot list resource "pods"

![image.png](/img/posts/K8scerts/noperm.png)


## 4. Role-Based Access Control (RBAC)

Once a user is authenticated, Kubernetes verifies what actions that user can perform using Role-Based Access Control.

![image.png](/img/posts/K8scerts/rbac.jpg)

### 4.1 RBAC Core Concepts

**Role**: Defines permissions for resources within a single namespace.

**ClusterRole**: Defines permissions across all namespaces.

**RoleBinding**: Binds a Role to a user or group within a namespace.

**ClusterRoleBinding**: Binds a ClusterRole to a user or group globally.

### 4.2 Creating a Custom Read-Only Role

Create a file named hamza-role.yaml:

![image.png](/img/posts/K8scerts/roles.png)

Apply it:

```bash
kubectl apply -f hamza-role.yaml
```

### 4.3 Binding the Role to the User

Create a file named hamza-rolebinding.yaml:

![image.png](/img/posts/K8scerts/rolebinding.png)

Apply it:

kubectl apply -f hamza-rolebinding.yaml

### 4.4 Testing Permissions

Switch to the context of the new user:

kubectl config use-context hamza-context

Allowed actions:

```bash
kubectl get pods
kubectl get deployments
kubectl get services
```

Denied actions:

kubectl delete pod <pod-name>

Expected output:

User "hamza" cannot delete resource "pods"

### 4.5 Verifying RBAC Permissions

You can query what the user can do:

```bash
kubectl auth can-i list pods
kubectl auth can-i delete pods
kubectl auth can-i create deployments
```

Output:

yes
no
no

![image.png](/img/posts/K8scerts/cani.png)

## 5. The system:masters Group

The default ClusterRoleBinding “cluster-admin” grants full control to the group ***“system:masters”***:

```bash
kubectl describe clusterrolebinding cluster-admin
```

**Subjects:**
Group system:masters

Any user certificate with **O=system:masters** automatically inherits cluster-admin rights. This is why the Minikube default user is an administrator. Custom users, such as “hamza” with O=dev-team, must receive explicit RBAC bindings.


## 6. Layers of Kubernetes Security

## Kubernetes Security Layers

Kubernetes security is structured as a **multi-layered defense model**, where each layer builds upon the trust and enforcement mechanisms of the previous one.  
This layered approach ensures that even if one layer is compromised, the others continue to provide protection.

| Layer | Purpose | Mechanism |
|--------|----------|------------|
| **Transport Security** | Protects communication between components | TLS certificates ensure encrypted and authenticated exchanges |
| **Authentication** | Verifies identity | Certificates, service account tokens, or external OIDC providers authenticate users and components |
| **Authorization** | Defines permissions | RBAC, ClusterRoles, and RoleBindings determine which actions are permitted |
| **Admission Control** | Enforces policy and compliance | Admission controllers validate or modify API requests before resource creation |
| **Runtime Security** | Secures workloads during execution | Network Policies, Pod Security, and sandboxing mechanisms restrict pod behavior |

Each layer strengthens the next, forming a **complete**


## Conclusion

Kubernetes security begins with cryptographic identity. Certificates secure communication between the control-plane components, while RBAC governs what authenticated users can do.

The Certificate Authority forms the root of trust. Certificates authenticate machines and users. The kubeconfig file binds those identities to clusters and namespaces. RBAC enforces least-privilege access across all namespaces.

By generating your own certificates, adding them to kubeconfig, and defining custom roles, you gain full visibility into how Kubernetes authenticates and authorizes every interaction. Understanding this trust chain is essential for operating Kubernetes securely in production environments.