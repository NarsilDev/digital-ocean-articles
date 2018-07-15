## Introduction

[Kubernetes](https://kubernetes.io/), the open-source container orchestration platform, is steadily becoming the preferred solution for automating, scaling and managing high availability clusters. As a direct result of its increasing popularity Kubernetes security has taken great relevance.

Generally speaking, a cluster is as secure as the infrastructure it belongs to. That means you need looking at the bigger picture, not only Kubernetes but also the elements affecting it starting with the cloud provider and moving through all the intermediate components until reaching the application itself. Main attacks will come from both ends, some will target the application in order to gain access to containers, nodes and finally the master node, but others will aim the master node from the beginning with the purpose of taking control over the cluster.

![Typical Attack Vectors](https://i.imgur.com/B19vDAk.png)

Considering the myriad of moving parts involved and the different deployment scenarios, securing Kubernetes may seem like a daunting task. The goal of this article is providing a solid security foundation for your cluster, a starting point resembling "Initial Server Setup" guides but with a Kubernetes-centric scope. In order to achieve this ambitious objective, the process will be split into more manageable steps:

* **Cloud Security & Cluster Authentication:** steps from one to five group the best security practices affecting the entire cluster and common to all kind of deployments independently of their particular usage, its main focus is protecting  from direct Internet attacks.
* **Role Based Permissions & Network Policies:** steps from six to eight dives into Kubernetes RBAC and network policies as a defense mechanism against unauthorized users.
* **Kubernetes Admission Controllers:** this step enforces using resources limitations and pod security policies which prevent privilege escalation attacks. 
* **Application Level Security:** this final step covers the generalities about securing your application and avoiding it compromising the cluster.

By the end of the article, you will hopefully have enough information to visualize Kuberbetes security in a broader context, giving you a unique perspective for complementing these security solutions or adapt them to tackle the challenges of your particular scenario. 

## Prerequisites

In order to complete this tutorial you will need:

* Two Droplets with a minimum of 4 GB of RAM running Ubuntu 16.04 configured using [this initial server setup tutorial,](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04) including a sudo non-root user and a firewall.
* A local client PC with Internet connection running Ubuntu 16.04 or Ubuntu 18.04.
* Docker-ce installed in both ends (client and server). You will find detailed instructions for Docker installation in our guide [How To Install and Use Docker on Ubuntu 16.04.](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-16-04)

https://www.digitalocean.com/community/tutorials/an-introduction-to-kubernetes

## Step One - Cloud Provider

As mentioned earlier in the introduction, securing your cluster starts with your cloud provider. Choosing the ideal cloud provider for your cluster it's a mix of personal preferences, budget, company policies, and practical security considerations. Kubernetes documentation provides a section dedicated to the different options regarding [picking the right solution.](https://kubernetes.io/docs/setup/pick-right-solution/)

This guide assumes you are using a universal custom solution for your Kubernetes deployment. From a security perspective, the factors you should consider for your cloud provider are:

* **Ability to configure private networks:** this is one of the most important security aspects. A cluster is basically a group of hosts that constantly interchange data over the network. Most of this traffic should not reach the Internet, thus, one effective strategy is using a private network. This feature can be activated [during Droplet creation](https://www.digitalocean.com/community/tutorials/how-to-create-your-first-digitalocean-droplet#step-6-%E2%80%94-selecting-additional-options). Depending on your project scope you can even spread your cluster among several regions, in which case a good security practice is using [point-to-point VPNs](https://www.digitalocean.com/community/tutorials/how-to-create-a-point-to-point-vpn-with-wireguard-on-ubuntu-16-04) between data centers to connect Master nodes.
* **High-Level Monitoring and Alerting:** the importance of proactive monitoring will be mentioned several times during the course of this guide. Ideally, your preemptive system should detect anomalies and send alerts promptly, before malicious scripts can do any harm. With that in mind, an extra layer of [monitoring](https://www.digitalocean.com/community/tutorials/an-introduction-to-digitalocean-monitoring) at the cloud level is a big plus.
* **High-Level Firewall:** using a [cloud firewall](https://www.digitalocean.com/community/tutorials/an-introduction-to-digitalocean-cloud-firewalls) supplements your host-based solution and also adds an independent layer of protection against several types of attacks. The cloud firewall is also very handy if you are not using a private network for cluster communications because many ports would be exposed to Internet otherwise. It's a best practice to open only the strictly necessary ports at this level (SSH and application ports).

Summing up, this high-level layer groups important security improvements that are many times neglected even when they are the foundation of a secure Kubernetes cluster.

## Step Two - Node Hosts Security

Nodes are the single most important component of the cluster. Because of how nodes interconnect within each other if any node is compromised the whole cluster could be at risk. Securing a node is not different to securing a server, you need to pay attention to factors like:

* **Base operating system:** you can install Kubernetes API server in any Linux distribution. But depending on the scale of your project you might want to use a specialized operating system designed with containers in mind. There are several options ranging from [RancherOS](https://rancher.com/rancher-os/), [CoreOS](https://coreos.com/), [Ubuntu Core](https://www.ubuntu.com/core), [Mesosphere](https://mesosphere.com/) and many others. For the purpose of this guide Ubuntu server 16.04 LTS will be used in all nodes.
* **Operating system updates:** depending on your choice of OS you might want to automate updates, especially security fixes that could compromise the node.
* **Enabling kernel-based security modules:** unlike virtual machines, containers share the host's kernel. That's why it's important to enforce kernel-based security modules because any exploit on the kernel can potentially compromise the entire node. Once again, depending on the OS you have many options: SELinux and AppArmor are among the most known but you could also use Seccomp security profiles in your Docker containers for added isolation. 
* **Intrusion Detection Systems (IDS):** you may use a traditional host-based solution or you can use a container based IDS that monitor the hosts as well as your cluster. What is important is to implement an intrusion detection system (or similar solution).
* **Node hosts monitoring and log audit:** a proactive monitoring of your server logs is crucial to prevent possible failures in your cluster. Suspicious activity (like high CPU load) could be a sign of a denial of service attack but are not the only types of attacks that can be prevented with a good monitoring. More specialized tools can detect containers trying to spawn a shell or writing to host's file system without needing too.
* **Firewall:** configuring your firewall is highly dependent on your use case. For instance, you may use a private network to connect your nodes as mentioned in the previous section or maybe your project requires a separate **etcd** cluster. In general, the best practice remains the same, once you determine the cluster topology never expose a port which doesn't need to be open.

Going a step further from the explained above, a solution for minimizing hosts vulnerability is not exposing them at all. You can accomplish that goal using a **Bastion Host**, a special node outside of the cluster using a security-hardened operating system that faces the Internet but also belongs to the cluster's private network. The Bastion Host approach can manage and filter traffic not only for SSH sessions but also Kubernetes remote sessions. This solution is ideal for private clusters because you are avoiding direct node access.

![Bastion Host](https://i.imgur.com/zzbQOC5.png)

Up to this point, we have examined the cloud layer and the host-centric security layer. All that before even start looking into Kubernetes security specifics. The logic behind that is protecting the cluster from port scans, unauthorized SSH access, and denial of service attacks just to mention a few. The following sections will introduce you to Kubernetes security aspects but still within the global context of the cluster.

## Step Three - Kubernetes Cluster Authentication

Before starting this section, its necessary to clarify the difference between **authentication** and **authorization** because this often leads to confusion. When you deploy a new Kubernetes cluster you are also installing by default an SSL certificate and key-pair which allows authenticating your user with the Kubernetes API. That only opens the cluster's door but doesn't grant the user any permissions because privileges are assigned depending on user role later during authorization stage. This is basically a two-step process, for every call sent to the Kubernetes API server the user must successfully authenticate and then pass through the authorization check before being able to do anything.

![Kubernetes Authentication](https://i.imgur.com/a0uzIxE.png)

This section will concentrate on the authentication phase, the main entrance of your cluster. Kubernetes offers several authentication methods:

* **Client Certificates:** as said earlier, Kubernetes installs a default self-signed Certificate Authority (CA) and private key when is deployed. Using the same **CA** you can generate additional certificates and keys for your new users.
* **Bearer Tokens:** another supported authentication method is using signed bearer tokens, which can be supplied specifying a file, or by a service that supports webhook tokens (like GitHub) or directly through the command line. 
* **Authenticating Proxy:** Kubernetes also allows using an intermediate proxy for authentication purposes. One popular example is using [OAuth](https://oauth.net/) compatible services for user management and validation but basically, you can use any user authentication platform able to generate the appropriate credentials for your cluster, even a properly configured [Nginx](https://www.nginx.com/) proxy could be used for this task. 
* **HTTP Basic Auth:** finally, you could authenticate by means of basic user/password static files or passing those arguments through the command line. 

An important security aspect when evaluating which method suit your needs is the concept of **user** from a cluster perspective. Kubernetes does not have objects which represent normal user accounts, it only manages service accounts. This directly impacts your user management, because there is no built-in procedure for banning normal (human) users. An additional security aspect to consider is that basic auth credentials, as well as tokens, last indefinitely this could be a huge drawback from a security point of view. 

The above force you defining an authentication strategy beforehand. For the purpose of this guide, SSL/TLS certificates will be used as the authentication method. Once you have a better understanding of authentication principles a final wrap up of the benefits and drawbacks of all of them will be made.

### Authenticating using Contexts

A Kubernetes context is nothing more than a way to describe how to authenticate a particular user into a specific cluster-namespace combination. Defining contexts is very easy using Kubernetes multi-purpose command line tool [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/). All important information regarding authentication is stored in its configuration file located by default at `~/.kube/config`. This file is commonly known as *kubeconfig* and you can view its content from cluster the Master node.

Start a new SSH session in the cluster Master node:

```command
[environment local]
ssh ubuntu@<^>master_ip<^>
```

Print out the kubeconfig using the command:

```command
[environment second]
kubectl config view
```

The output will be similar to the following:

```
[secondary_label ~/.kube/config]
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: REDACTED
    server: https://<^>master_ip:port<^>
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```

* **apiVersion:** is a mandatory field in all Kubernetes definition files describing the version used for the conventions.
* **clusters:** this section provides the specifics about the cluster connection: the public Certificate Authority, the cluster public IP address (including connection port), and the cluster name. This simplified version only define one cluster but you can use the same file for several clusters each one with its own connection settings.
* **contexts:** normally, you will find three parameters in this section:
    - **cluster:** sets the cluster you want to connect with. In the previous section, you entered the required information that's why is called by name.
    - **namespace:** in Kubernetes [namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) are virtual clusters used mainly to give you more control over workgroups or projects assets. Users confined to one namespace can't see or use any resource in other namespaces. When no namespace is defined the API assumes that you are connecting to the *default* namespace created during cluster deployment.
    - **user:** is the name of the user that will connect to this particular cluster-namespace combination.
    - **name:** the context name.
* **users:** includes the username and unique credentials. As you will learn shortly, the credentials may vary depending on the authentication method used.

From the output above is clear that **kubernetes-admin** authenticates in the Kubernetes cluster using SSL/TLS certificates. This configuration was done during the cluster bootstrapping using `kubeadm`, but what happens if you need adding more information or modifying these settings?

Let's start creating a new namespace:

```command
[environment second]
kubectl create namespace dmz
```

List your namespaces running the following command:

```command
[environment second]
kubectl get namespaces
```
You will see the an output similar to the following:

```
[secondary_label Output]
NAME          STATUS    AGE
default       Active    1d
<^>dmz<^>           Active    4s
kube-public   Active    1d
kube-system   Active    1d
```

Now you have two namespaces to play with, the **default** namespace created during cluster deployment and the recently added **dmz**. The other two namespaces **kube-public** and **kube-system** are reserved and should remain untouched. In order to avoid confusions during this guide, change the default context name (kubernetes-admin@kubernetes) to something more intuitive, like *kubeadmin-default*. To do so run the following the command:

```command
[environment second]
kubectl config rename-context kubernetes-admin@kubernetes kubeadmin-default
```

Now, you need adding the **dmz** namespace to kubeconfig, do it running the following command:

```command
[environment second]
kubectl config set-context kubeadmin-dmz --namespace=dmz --cluster=kubernetes --user=kubernetes-admin
```

Check again the kubeconfig to confirm all changes using the same command than before:

```command
[environment second]
kubectl config view
```
The resulting output should be similar to this one:

```
[label ~/.kube/config]
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: REDACTED
    server: <^>master_ip:port<^>
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: <^>kubeadmin-default<^> 
<^>- context:<^>
    <^>cluster: kubernetes<^>
    <^>namespace: dmz<^>
    <^>user: kubernetes-admin<^>
  <^>name: kubeadmin-dmz<^>
current-context: <^>kubeadmin-default<^>
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```

From now on,  switching between contexts is as easy as running the command:

```command
[environment second]
kubectl config use-context <^>context-name<^>
```

Let's review the most important aspects of Kubernetes authentication so far:

- All requests made to the Kubernetes API must be authenticated, even services.
- Kubernetes does not define an object or resource for normal users, just for service accounts, so users management is independent of Kubernetes.
- The kubeconfig stores all sensitive and relevant data regarding users authentication in the form of contexts.

Each time you use `kubectl` it looks for kubeconfig in its default location, which can be override using the `--kubeconfig` flag or `KUBECONFIG` environment variable. That is possible because kubeconfig is a **portable** configuration, is only a descriptive file in YAML format indicating the necessary information for authenticating the user. This offers a huge flexibility, because you can use different kubeconfig files or, better yet, you can **export** its settings to a remote client.

### Remote Cluster Authentication using SSL/TLS Certificates

In the previous section, you learned that kubeconfig is a portable file allowing remote `kubectl` connections. You may wonder if such approach worth the effort, considering that you already have a  secure SSH tunnel to your master node. The answer is simple, from a security perspective connecting via SSH with your Master node is far from ideal because you gain access to the entire host operating system not only Kubernetes. A selected group of administrators should use SSH sessions and even those administrators should prioritize remote `kubectl` connections over SSH sessions whatever possible.

Fortunately, setting up `kubectl` remote access couldn't be easier. Assuming the local client machine (named **manager** for convenience) already have `kubectl` installed, the only thing you need is transferring the `~/.kube/config` from your Master node.

1. If your session in the Master node is still active go directly to step 2, if not, establish a SSH connection again using the same command as before:

    ```command
[environment local]
ssh ubuntu@<^>master_ip<^>
    ```

2. Get an output of the kubeconfig in raw format (unfold secrets) using the following command:

    ```command
[environment second]
kubectl config view --raw
    ```

3. Copy the entire content of the kubeconfig, it's a long output now that certificates and private key are visible, be sure to grab it all, and then close the SSH session pressing **Ctrl+D**.

4. Create a new `~/.kube/config` in your  local client and paste the **entire text**, once again double check you didn't miss any portion.

    ```command
[environment local]
mkdir ~/.kube
nano ~/.kube/config
    ```

5. Save and close the file and that's it! You have a local kubeconfig with all relevant information, including cluster's CA and **kubernetes-admin** credentials. Check your connection running from your local machine:

```command
[environment local]
kubectl cluster-info
```

You will get as a response the cluster-specific information. similar to the shown below:

```
[secondary_label Output]
Kubernetes master is running at https://<^>master_ip<^>:6443
KubeDNS is running at https://<^>master_ip<^>:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

<$>[warning]
**Warning:** your local machine is now connecting with **kubernetes-admin** superuser. Keep kubeconfig content safe, as you would do with any file including SSL certificates.
<$>

Up to this point, you have a local client connecting with the cluster using a secure SSL/TLS connection. The next logical step is allowing new users to do the same which is precisely the topic of the next section.

### Authenticating New Users using SSL/TLS Certificates

Adding more users to the cluster is not a trivial task. As discussed earlier, Kubernetes API is unaware of user management.  This section will guide you through the process of creating two new remote users: a regular namespace administrator **adm** and a normal user **sammy**, each one having independent kubeconfig files.

1. One more time, you'll need an active SSH session on your Master node connect using the following command:

    ```command
[environment local]
ssh ubuntu@<^>master_ip<^>
    ```

2. By default, all cluster certificates are stored in `/etc/kubernetes/pki/` (including Certificate Authority public certificate and private key). The objective is transferring `ca.crt` and `ca.key` to a safe location on your local machine, this guide will assume that location is `~/certs`. Using `cat`print out the each certificate and copy its contents into a temporally file/notepad:

    ```command
[environment second]
sudo cat /etc/kubernetes/pki/ca.crt
sudo cat /etc/kubernetes/pki/ca.key
    ```

3. Once you have the information copied close the SSH session and create the new directory in the local machine:
    ```command
[environment local]
mkdir ~/certs
    ```

4. Now paste the content of the public certificate and the private key in their corresponding files:

    ```command
[environment local]
nano ~/certs/ca.crt
nano ~/certs/ca.key
    ```

5. After saving the files, you will need changing permissions for both of them running the following command:

    ```command
[environment local]
chmod 644 ~/certs/ca.crt && chmod 600 ~/certs/ca.key
    ```

6. Granting `root` ownership to the certificates will bring them enhanced security, do it running the following command:

    ```command
[environment local]
sudo chown root:root ~/certs/ca.key && sudo chown root:root ~/certs/ca.crt
    ```

7. Now that you have the certificates at hand you can proceed creating users private keys using the following commands:

    ```command
[environment local]
openssl genrsa -out ~/certs/sammy.key 4096
openssl genrsa -out ~/certs/adm.key 4096
    ```

8. Now generate the certificate signing requests specifying in the subject the user name as common name (CN) and the user group as organization (O):

    ```command
[environment local]
openssl req -new -key ~/certs/sammy.key -out ~/certs/sammy.csr -subj "/CN=sammy/O=developer" 
openssl req -new -key ~/certs/adm.key -out ~/certs/adm.csr -subj "/CN=adm/O=administrator"
    ```

9. Sign the certificates using the cluster CA and private key and assign them an expiration date of 90 days using the following commands:

    ```command
[environment local]
sudo openssl x509 -req -in ~/certs/sammy.csr -CA ~/certs/ca.crt -CAkey ~/certs/ca.key -CAcreateserial -out ~/certs/sammy.crt -days 90
sudo openssl x509 -req -in ~/certs/adm.csr -CA ~/certs/ca.crt -CAkey ~/certs/ca.key -CAcreateserial -out ~/certs/adm.crt -days 90
    ```

10. Now add the new credentials to the local `kubeconfig` running these commands:
    
    ```command
[environment local]
kubectl config set-credentials sammy --client-certificate=/home/<^>linux-user<^>/certs/sammy.crt  --client-key=/home/<^>linux-user<^>/certs/sammy.key 
kubectl config set-credentials adm --client-certificate=/home/<^>linux-user<^>/certs/adm.crt  --client-key=/home/<^>linux-user<^>/certs/adm.key
    ```

11. The configuration file now holds the credentials of all users, including **kubernetes-admin** what would be a problem, because the last thing you want is distributing a configuration with a superuser profile. Fortunately, using `kubeadm` you can generate an independent kubeconfig for each user running the following command:

    ```command
[environment local]
sudo kubeadm alpha phase kubeconfig user --client-name=adm --cert-dir="/home/<^>linux-user<^>/certs"
    ```

<$>[note]
**Note:** the command requires the **absolute path** to the certificates directory to function properly.
<$>

12. Copy the resulting output (starting from **apiVersion**) and then create a new file for the user, for example: 

    ```command
[environment local]
nano ~/.kube/config-adm
    ```

13. Paste the `kubeadm` output into `~/.kube/config-adm`, then save and close the file and repeat the procedure for the other user:

    ```command
[environment local]
sudo kubeadm alpha phase kubeconfig user --client-name=sammy --cert-dir="/home/<^>linux-user<^>/certs"
    ```

14. Copy once again the resulting output (starting from **apiVersion**) and create a new `sammy` configuration file running:

    ```command
[environment local]
nano ~/.kube/config-sammy
    ```

15. Paste the  `kubeadm` output into `~/.kube/config-sammy`, then save and close the file.

These files are ready for distribution because they only contain the SSL/TLS credentials of their respective user. Later on, once the files are saved on their corresponding client machine, you may want to rename `config-adm` and `config-sammy` to the default `config` filename to avoid using `--kubeconfig=<^>config-name<^>` flag each time you issue a command.

<$>[note]
**Note:** using `kubeadm alpha phase kubeconfig user` is an alpha feature, and will only generate the kubeconfig template, you must create the file and save it into user's `~/.kube/` before adding more namespaces to user's profile through `kubectl`.
<$>

For simplicity, instead of using three independent client machines this guide will keep using only the **manager** client but with all contexts properly configured in the same kubeconfig. Since the users' credentials are already included you only need adding **default** and **dmz** namespace information using the following commands:

```command
[environment local]
kubectl config set-context adm-dmz --namespace=dmz --cluster=kubernetes --user=adm
kubectl config set-context adm-default --namespace=default --cluster=kubernetes --user=adm
```

You can also include **sammy** contexts using similar commands:

```command
[environment local]
kubectl config set-context sammy-dmz --namespace=dmz --cluster=kubernetes --user=sammy
kubectl config set-context sammy-default --namespace=default --cluster=kubernetes --user=sammy
```

Once you are done, test **adm** remote connection in **default** namespace trying to list the pods running the following commands:

```command
[environment local]
kubectl config use-context adm-default
kubectl get pods
```

You will get an error similar to the shown below:

```
[secondary_label Output]
Error from server (Forbidden): pods is forbidden: User <^>"adm"<^> cannot list pods in the namespace "default"
```

Let's take a moment to understand what happened. You can assert that you successfully established an SSL/TLS connection with the remote cluster and also that the API server authenticated **adm** because of the error message stating clearly that the user is restricted to perform the request. The API denied listing pods because **the user is not yet authorized to do so**. Authorization will be covered in another section, but for now, the goal of authenticating remote users was successful. Switch back to **kubeadmin-default** context running the following command:

```command
[environment local]
kubectl config use-context kubeadmin-default
```

As you can see, authenticating using SSL/TLS certificates is truly straightforward. Its key advantage is that you don't need to modify the API server settings. The default installation uses certificates, you are just adding more users by means of signing their own certificates with the same CA. From a security point of view, that's a major benefit because you don't need direct access to the cluster in order to grant authentication for new users, you only need to keep your cluster credentials safe.

However, there are also two major drawbacks. As mentioned several times now, Kubernetes does not have a built-in mechanism for managing normal users. If you have a large user base the process of generating and managing certificates could become a challenge. The other disadvantage is that all certificates will remain active unless you manually remove them and restart the API service. That's why is highly recommended to add an expiration date for each certificate. It adds the nuisance of generating new certificates but enforces security because you can set different expiration times depending on your own convenience. For example, you can create credentials for a superuser lasting only one day if you need to. Then you can clean-up certificates during next scheduled cluster maintenance.  

Now that you have a better knowledge of authentication principles its time to evaluate the rest of the methods available.

### Remote Cluster Authentication using other methods

As said in the previous section, the default installation uses certificates for authentication, you can change those setting editing `/etc/kubernetes/manifests/kube-apiserver.yaml` in the master node and adding or modifying the corresponding flag:

* **--client-ca-file=**<^>/path/to/CA<^>: this option points to the desired Certificate Authority that will be used to validate client certificates presented to the API. You can use the default CA created during installation but can also add a different one if you need to.
* **--token-auth-file=**<^>/path/to/TOKEN_FILE<^>: this option tells the API where it can find the bearer tokens csv file. This file must have a minimum of three columns: token, user name and user UID followed by optional group names. Each user must have its own line within the file.
* **--basic-auth-file=**<^>/path/to/PASSWORD_FILE<^>: this flag enables the option for using static username/passwords for authentication. It works in the same way as the bearer token option, you need to provide a csv file with three or four columns that include: password, username, user id and optionally groups.
* **--oidc-issuer-url / --oidc-username-claim / --oidc-client-id:** an additional authentication method is through OAuth2 providers (Azure Active Directory, Salesforce, Google). Is out of the scope of this guide explaining the complete configuration process, but basically, the authentication providers are responsible of supplying a special kind of token called JSON Web Token (JWT) which includes the relevant user information. That token is then used as the bearer token for authenticating in Kubernetes API.  

You can use different authentication methods simultaneously, but keep in mind what was mentioned in the previous sections, some changes to `kube-apiserver.yaml` may need a complete service restart.

Summing-up, when analyzing all the information one can conclude that using an authentication proxy (for example, an OAuth2 service) is possibly the best solution to manage a large user base at the cost of additional complexity during the cluster set up. On the other hand, manual management of user tokens or password files may seem highly inefficient. But even those methods could find a good use case, for example, for demonstration purposes you can supply a preconfigured username and password to your clients in order to access a restricted demo in a cluster or namespace specially configured for that goal. Finally, you have an intermediate solution when using SSL certificates because properly managing their expiration dates you can establish a flexible security policy that suits many use cases.

## Step Four - Backend access (**etcd** integration).

Kubernetes uses an **etcd** backend for storing all cluster data, including sensible information like the TLS credentials used for authentication. This makes **etcd** a desirable target for attackers and thus is a high priority to keep it safe.

The default `kubeadm` installation deploys a single pod running the **etcd** server in the master node which means that the communication between the master and the backend is local and there is no implicit risk involved. But what happens when you need a High Availability Cluster? 

A production Kubernetes HA Cluster requires an independent **etcd** backend, usually another cluster with three or more nodes. In that case you must enforce the security of your backend implementing:

* A restrictive firewall policy that only allows communications between the Kubernetes API server and the **etcd** cluster. If the **etcd** cluster is in the same datacenter as the API server, a private network is strongly recommended.
* Limit the read and write access of other cluster components to the **etcd** cluster. Ideally, only the API server should access the **etcd** cluster.
* Strong mutual TLS authentication between both ends (API server and **etcd** server).

The last point, strong mutual TLS authentication, is arguably the most important but its configuration, as well as the creation of the required certificates, belongs to the **etcd** side. From Kubernetes perspective, you should avoid applications pretending direct access to **etcd** backend whatever possible.  

## Step Five - Cluster monitoring & Audit logging.

From a security viewpoint, monitoring any suspicious activity in your cluster is a top priority task. The high-level monitoring suggested at the beginning of this guide offers a panoramic view of your node's resources, what is very helpful, but does not offer any information regarding containers and other cluster specific processes. This section explores the alternatives at hand in order to collect the necessary monitoring metrics.

### Kubernetes Monitoring

The term monitoring has evolved from old host-centric concept moving forward to the current trend of Orchestrated Containerized Infrastructure. 

![Kubernetes Monitoring](https://i.imgur.com/TS7AFaN.png)

As you can see there is a lot more information to process. Let's break down the new monitoring landscape by layer:

* **Host-level:** in Kubernetes terms are the nodes part of your infrastructure. The underlying Operating System is usually the primary source of data regarding the node status through using host-based or cluster-based agents to gather this information. If you are following the recommended setup, the cloud-enabled monitoring and alerting system should be already in place gathering information from your nodes.
* **Cluster-level:** encompasses all Kubernetes related information such as logs, events, activity, workloads but also include container related and host related information collected from the API server and cluster specific agents. This task is performed by cAdvisor that comes enabled by default with each running Kubelet.
* **Container-level:** comprises the information reported by your container engine (Docker or rkt). 
* **Application-level:** inside each container is your actual application which most probably has its own reporting engine, logs, and metrics. Once again, you have several options to manage this data. You can send all application related information directly to an independent third-party back-end monitoring system, or you may use a container side-car to collect the relevant information and then report it to the Docker engine and/or Cluster-level monitoring solution.

It's important noticing that metrics, like host workload, is collected by most layers independently. Since Kubernetes uses different time intervals for data acquisition and different approaches regarding what information is relevant and what's not (rules and filters), the raw data from the different monitoring layers will not always yield the same results. Since your goal is proactively monitoring looking for malicious activity, it's very important to determine what layer has a greater impact on the infrastructure. Knowing your application is fundamental because depending on its particular requirements, you could focus on host-wide parameters or more granular container metrics. 

Hand to hand with monitoring is logging audit. Your monitoring system is in charge of detecting parameters outside its normal values and then trigger an automatic alert. But in order to evaluate the situation, the most valuable information is in your logging system.

According to the explained above, you may ask what would be the best strategy for a new Kubernetes deployment?

* **Logging management:** in order to manage the huge amount of log information coming from nodes, clusters, container engine and even applications an accepted solution is using the ELK Stack or any of its variations. Specifically speaking about cluster-level logs, although Kubernetes does not provide a native solution it offers built-in support for [Elasticsearch](https://www.elastic.co/) logging end-point and [Fluentd](https://www.fluentd.org/) agent. That means that you could deploy a central logging system using Elasticsearch, Fluentd, and [Kibana](https://www.elastic.co/products/kibana) (EFK Stack).
* **Cluster Monitoring:** a popular solution to monitoring your cluster infrastructure is using [Prometheus](https://prometheus.io/) and [Grafana](https://grafana.com/). However, it's not the only solution you have available. Kubernetes uses cAdvisor embedded into Kubelets to gather container metrics and Heapster to report cluster-level data and events. You can also use them in conjunction with [InfluxDB](https://www.influxdata.com/) and Grafana if that suits you better. 

Once you set up your logging and monitoring systems you can configure them to detect anomalous activity in your nodes, pods, and containers, think of it as your Events Alerting System.

Up to now, security measures were focused on the global context of the cluster.  A brief summary of what was accomplished so far is:

- Attackers aiming at your infrastructure will face at least two firewall barriers, one independent of the other.
- The intrusion detection system, high-level monitoring, and cluster-level monitoring bring you an alerting system that opens the possibility to block several attacks in real time.
- Nodes remote SSH access as well as `kubectl` remote sessions can be highly restricted through the use of a Bastion Host approach.
- Cluster authentication is accomplished using secure SSL/TLS certificates (or alternative secure methods).
- The **etcd** backend is secured either because is running from the same master node or because is isolated and restricted from the Internet in a separate cluster.

As you can see, many external vulnerabilities are already prevented in the global scope of the cluster. The next sections will cover the security aspects related to users that were able (legitimately or not) to authenticate into the cluster.

## Step Six - Authorization and Role Based Access Control (RBAC)

Up to now, the security approach has been focusing on the global context of the cluster. You have minimized the attack surface targeting your infrastructure as well as cluster authentication. The next sections cover Kubernetes internals mostly including the best practices for preventing the cluster from being hacked down the road. 

Authorization was briefly explained as a way of granting permissions based on user roles. Kubernetes uses indeed a Role Based Access Control (RBAC) allowing great flexibility when handling permissions.  

![Kubernetes Role Based Access Control](https://i.imgur.com/czcQWPe.png)

From the image above you can infer that depending on the attributes read by the API server, the authorizer determines if the call is rejected or not. The authorization process is divided into two fundamental stages: 

* **Reviewing Request Attributes:** once the API call passes the authentication check the server gathers its attributes: user, group, request path, request verbs, namespace, resources, API group. Those attributes are necessary in order to answer *who* is the entity making the request, *what* it wants to do, *where* it wants to execute it and *what* are the resources affected.
* **Evaluating Attributes Against All Declared Policies:** in Kubernetes terms, authorization policies are called roles. The RBAC module is responsible for finding the user/service *bindings* and deciding if the request complies with the assigned *roles* for the particular combination of actions, resources and namespace specified.

That leads to two key concepts: *roles* and *bindings*. RBAC default is denying all requests to the API server meaning you need to assign permissions depending on your needs. Each role describes what verbs (create, get, delete, list, etc) are allowed for determined resources (pods, nodes, secrets, namespaces). Consider roles as a convenient group of permissions. Once you create the roles, you need to bind them to users or service accounts that need authorization for using resources on a particular namespace or for the whole cluster.

Formally speaking, Kubernetes RBAC recognizes four top-level types of objects for managing permissions:

* **Role**: is a set of permissions that apply to a single namespace.
* **ClusterRole**: is also a set of permissions but their scope is the entire Cluster (they are not restricted to one namespace, they are global).
* **RoleBinding**: grants permissions within a single namespace to users, groups or service accounts.
* **ClusterRoleBinding**: grants cluster-wide permissions to users, groups or service accounts. 

The best way to understand the concepts behind access control is returning to our user **adm**. He was able to authenticate using his SSL/TLS credentials, but when he tried performing an action (verb) the RBAC module forbid it. He only needs a role binding. You can corroborate his lack of permissions running from the **manager** local client the following command:

```command
[environment local]
kubectl auth can-i get pods --namespace default --user=adm
```

The simple answer on the terminal will be *no*. The command `kubectl auth can-i` evaluates the attributes and then verifies the associated roles for the user. Over again on  **adm**, let's say that you only want to give him permissions to fulfill the request above and list pods. Kubernetes RBAC authorizer has a default set of  cluster roles pre-configured, list them using the following command:

```command
[environment local]
kubectl get clusterrole
```

You will see a long list including system group roles. However, there are four universal and very convenient cluster roles defined:

- **view:** grants permissions to see non-critical resources. This is the least permissive ruleset, and the safest to implement as the default.
- **edit:** inherits the role from the *view* cluster role and grants basic permissions for executing simple actions like creating, deleting, listing and watching pods as well as other common resources. In most cases, this level is adequate for normal users within a namespace.
- **admin:** as expected, inherits the role from the *edit* cluster role and complements it with wider namespace permissions. This role is commonly used for namespace administrators. Due to its permissions level, you should be careful before granting this cluster role to any user.
- **cluster-admin:** grants access for administering the entire cluster. This is the most permissive cluster role and should be handled with extreme caution. In practice, any user or service with this authorization level is considered a superuser.

<$>[note]
**Note:** you can get a detailed description of permissions included in any cluster role using `kubectl describe clusterrole` <^>ClusterRole_name<^>
<$>

If you only need **adm** listing available pods then you can assign him the **view** cluster role running the command:

```command
[environment local]
kubectl create rolebinding adm-view-test --clusterrole=view --user=adm --namespace=default
```

You will see an output similar to this one:

```
[secondary_label Output]
RoleBinding.rbac.authorization.k8s.io "<^>adm-view-test<^>" created
```

Indicating that the new role binding was successfully added to the authorization group. You can now check if is working as expected using again:

```command
[environment local]
kubectl auth can-i get pods --namespace default --user=adm
```

It's very important noticing that **restarting the API server wasn't necessary** for authorizing the user. This is a key difference between authentication mechanisms and RBAC authorization: *you can dynamically change permissions.*

Revoking viewing access is very easy running the command:

```command
[environment local]
kubectl delete rolebinding adm-view-test
```

A short message will indicate that the role binding was deleted.

So far, the basics about Kubernetes authorization has been explained. Next sections will cover more specific usage of access control for users as well for applications.

### Customizing Authorization Roles

In the last section, you granted pre-configured view permissions to the test user **adm**. But, what happens if you need a role or cluster role more flexible?

Just as any other Kubernetes object, you can use a YAML file to describe it, in this case a role. Let's imagine you want a role that allows a determined user to view pods (only pods). The procedure is quite simple:

First create a new working directory for your objects definitions, for example `~/kube-security` :

```command
[environment local]
mkdir ~/kube-security
```

Next create the cluster role file, in this case called `pod-viewer.yaml`:

```command
[environment local]
nano ~/kube-security/pod-viewer.yaml
```

Paste into the file the following content:

```
[label pod-viewer.yaml]
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: pod-viewer
rules:
- apiGroups: [""] 
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

A closer look at the file above shows the structure for defining a cluster role:

* **kind:** indicates if you are creating a `Role` or `ClusterRole` object.
* **metadata:** in this case only includes the necessary Cluster Role name.
* **rules:** this section encompasses the main information that defines your rule: 
    - **apiGroups:** the modules where the rule applies, in this case, the wildcard refers to the core group.
    - **resources:** as the name implies is a list of resources affected by this rule, in this case only pods.
    - **verbs:** a listing of allowed actions. Remember that RBAC default is deny everything, this is an additive list of actions.

Once the file is saved you can create the new cluster-wide role running from the **manager** client the following command:

```command
[environment local]
kubectl create -f ~/kube-security/pod-viewer.yaml
```

Similar than before, you need biding the role to the user using the following command:

```command
[environment local]
kubectl create rolebinding adm-pod-view --clusterrole=pod-viewer --user=adm --namespace=default
```

Test the user authorization once again:

```command
[environment local]
kubectl auth can-i get pods --namespace default --user=adm
```

And that's all you need. The user is allowed to view pods in the default namespace. You may notice that all examples used a cluster role instead of just a role. Cluster roles are by definition more reusable because they grant permissions for the whole cluster but you can restrict them if necessary using a simple `RoleBinding` limiting the user access to a designated namespace. However, nothing stops you to create a custom-tailored role for a particular user (or application). 

Create another authorization file for the new role:

```command
[environment local]
nano ~/kube-security/default-pod-viewer.yaml
```

Paste the following role and save the file:

```
[label ~/kube-security/default-pod-viewer.yaml]
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: default-pod-viewer
rules:
- apiGroups: [""] 
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

As you see, this role grants the same permissions than the last one, being the difference its scope explicitly declared in the metadata section. You can create the role using the following command:

```command
[environment local]
kubectl create -f ~/kube-security/default-pod-viewer.yaml
```

You can also bind the role using a custom file. First create the new role binding:

```command
[environment local]
nano ~/kube-security/default-adm-pod-view.yaml
```

And now paste the following definition into the file:

```
[label ~/kube-security/default-adm-pod-view.yaml]
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: default-adm-pod-view
  namespace: default
subjects:
- kind: User
  name: adm
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: default-pod-viewer
  apiGroup: rbac.authorization.k8s.io
```

Notice that two new sections are present: **subjects** and **roleRef**. A subject can be either a user, a group or a service account. The second section references an already created role (or cluster role) by name and API group. Before applying this role delete the old one using this command:

```command
[environment local]
kubectl delete rolebinding adm-pod-view
```

And now create the new role binding, this time referencing the file containing the definition:

```command
[environment local]
kubectl create -f ~/kube-security/default-adm-pod-view.yaml
```

You can confirm the user permissions using:

```command
[environment local]
kubectl auth can-i get pods --namespace default --user=adm
```

Clean-up permissions deleting **adm** authorization again:

```command
[environment local]
kubectl delete rolebinding default-adm-pod-view
```

Let's review the key elements of K8s authorization so far: 

* **Permissions scope:** you can group your permissions defining a **Role** or a **ClusterRole**. The main difference between them is the cluster-wide scope of the **ClusterRole** opposed to the namespace restricted **Role**.
* **Permissions binding scope:** similar to the last point, if you confer a **ClusterRoleBinding** you are basically granting permissions access to all namespaces. On the other hand, a **RoleBinding** is by definition restricted to one namespace.
* **Roles definitions:** indistinctly if you are creating a role or cluster role, you need to define rules specifying the API group, resources affected by the rule and allowed verbs for the rule. You can always use pre-defined Kubernetes roles if that suits you.
* **Binding entities:** when defining a role or cluster role you need declaring in the **subjects** section the target entity (user, group, service account), the name of the entity and the API group.

The real power of the RBAC API starts when you fully understand its flexibility. Once roles are defined you can *dynamically* allow and deny permissions to resources.

### RBAC Security in Practice

In this section, you will use what you learned about RBAC to assign different permissions levels for remote users **sammy** and **adm**. The following table shows the desired result:

|       |   **dmz**  | **default** |
|-------|:----------:|:-----------:|---|---|
| **view**  | sammy, adm |    sammy    |
| **edit**  | sammy, adm |     adm     |
| **admin** |     adm    |     adm     | 

1. Grant the global **admin** role to the user **adm** running the following command:

    ```command
[environment local]
kubectl create clusterrolebinding adm-admin --clusterrole=admin --user=adm
    ```

2. Grant the `view` role to **sammy** in the **default** namespace using this command:

    ```command
[environment local]
kubectl create rolebinding sammy-view --clusterrole=view --user=sammy --namespace=default
    ```

3. Finally, grant `edit` level permissions to **sammy** in the **dmz** namespace using this command:

    ```command
[environment local]
kubectl create rolebinding sammy-edit --clusterrole=edit --user=sammy --namespace=dmz
    ```

As you can tell, what at first seemed difficult or that required a lot of time was accomplished running three commands. Assigning the cluster-wide role **admin** to **adm** was enough for enabling all precedent permission levels for that user. On the other hand, **sammy** needed namespace-specific permissions, therefore, it was necessary applying two independent role bindings.

From RBAC authorization viewpoint the work is done because the API server is aware of the new permissions. You can review them using the following commands:

```command
[environment local]
kubectl get clusterrolebindings
kubectl get rolebindings --all-namespaces
```

The security implications of role-based access control will be discussed in more detail in the coming sections, for now, is enough knowing that you have a namespace administrator and a normal user properly configured.

## Step Seven - Kubelet Authentication & Authorization

A [kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/) is an agent present in each node in charge of running containers inside the pods as well as ensuring that they are healthy. As discussed earlier, Kubernetes forces cluster users to authenticate and then go through authorization check in order to perform any action. But, depending on how you initially configured your cluster, you may be using insecure settings for the kubelets, authenticating with the **system:anonymous** user and authorizing everything because the `AlwaysAllow` flag might be enabled. 

From a security perspective kubelets, should pass the authentication and authorization checks same as normal users. If you followed the suggested method for cluster installation (`kubeadm` bootstrapping) then you already have the kubelets securely configured. However, if you want to check their current configuration is not hard to do it.

First start a SSH session in any of your nodes, this guide only uses one node called **worker** so connect to it using the usual command:

```command
[environment third]
ssh ubuntu@<^>worker_ip<^>
```

Now from the node run the following command:

```command
[environment thrid]
sudo cat /var/lib/kubelet/config.yaml
```

You will see a long output with a lot of information regarding the kubelet configuration, look for the **authentication** and **authorization** sections at the beginning of the file, they should be similar to the following:

```
[label /var/lib/kubelet/config.yaml]
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
```

Let's review how is configured the kubelet:

* **authentication:** the aforementioned **anonymous** user is disabled and SSL/TLS authentication is used instead. Take notice of the Certificate Authority, is the default CA created during installation, the same one used during the *authentication* section.
* **authorization:** the *Webhook* mode delegates authorization to the API server, in practice, request attributes determines permissions same as for normal users.

Summing up, the security wise configuration for kubelets is performing the authentication using SSL certificates and then pass a signed bearer token for authorizing each call. That's the default configuration used by `kubeadm`.

## Step Eight - Network Access Policies

Kubernetes uses the [NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) specification for declaring rules regarding how pods are allowed to communicate.  Possibly the best way to understand this resource is imagining it like a *virtual firewall for pods*. Setting up network policies allows you to **isolate** pod access not only to other namespaces but even within the same namespace.

![Isolated Pod](https://i.imgur.com/3VtkwF4.png)

Extending the similarity with firewalls, let's say you need a *deny all* behavior for all pods within a namespace.

 In that case, you can create a file for your network policy using your favorite editor:

```command
[environment local]
nano ~/kube-security/deny-all.yaml
```

And then copy and paste the following definition:

```
[label ~/kube-security/deny-all.yaml]
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

Save and close, then create the policy running from your local client the following command:

```command
[environment local]
kubectl create -f ~/kube-security/deny-all.yaml
```

You can list current network policies and confirm its creation running the following command:

```command
[environment local]
kubectl get networkpolicy
```

The output should be similar to this one:

```
[secondary_label Output]
NAME       POD-SELECTOR   AGE
deny-all   <none>         10s
```

All pods part of the namespace will now deny incoming and outcoming connections. 

Kubernetes **NetworkPolicy API** let you control permissions using:

* **Selectors:** you can use Kubernetes `podSelector` or `namespaceSelector` in conjuction with `matchLabels` and `matchExpressions` in order to define what pods or namespaces you are referring to.
* **Policy Type:** using the `policyTypes` argument you can specify if your rule is for `Ingress` or `Egress` type of connection.
* **IP Address:** you have the flexibility to restrict connections by IP blocks.
* **Ports:** using the `ports` argument you can specify what protocol and port is allowed.

Back to the `deny-all` policy, you are instructing the API to select all pods in the **default** namespace and apply ingress/egress rules. Since no rules are included all connections will be refused. That strategy leaves you in charge of deciding what to accept. Let's explore how network policies operate with an example:

1. Create in the **default** namespace  a disposable **busybox** container and start an interactive session using the following command:

    ```command
[environment local]
kubectl run default-test --rm -ti --restart=Never --namespace=default --image=busybox /bin/sh
    ```

2. Once inside, take note of the container IP address (`eth0` interface):

    ```super_user
[environment third]
ip a
    ```

3. Don't close the session yet. Open a new terminal window pressing **Ctrl+Alt+T** and launch a new interactive session inside another disposable **busybox** container but this time running in the **dmz** namespace using this command:

    ```command
[environment local]
kubectl run dmz-test --rm -ti --restart=Never --namespace=dmz --image=busybox /bin/sh
    ```

4. Now that you are inside the `dmz-test` container take note of its IP address:

    ```super_user
[environment fourth]
ip a
    ```

5. Next, try reaching the `default-test` container using the ping command:

    ```super_user
[environment fourth]
ping <^>default-test_IP_address<^>
    ```

6. You won't get a response because the entire **default** namespace and all its pods are currently **isolated**. To double check it, go back to the `default-test` session and try ping `dmz-test` container from there:

    ```super_user
[environment third]
ping <^>dmz-test_IP_address<^>
    ```

As expected, you won't get any response either. Cancel the ping request pressing **Ctrl+C** and then close `default-test` session pressing **Ctrl+D**. Also cancel the ping request in the `dmz-test` container pressing **Ctrl+C** but **keep the session active** for now.

After establishing that pods created in the **default** namespace are indeed isolated, the next logical step is allowing some connections.

Create a new network policy file:

```command
[environment local]
nano ~/kube-security/allow-test.yaml
```

And now copy and paste the following definition:

```
[label ~/kube-security/allow-test.yaml]
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-test
  namespace: default
spec:
  podSelector:
    matchLabels:
      status: isolated
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          status: allowed 
    - podSelector:
        matchLabels:
          app: my_app
  egress:
  - {}
```

The `allow-test` network policy applies some new rules in the **default** namespace:

- First, it isolates all pods with the label `status=isolated` created in the **default** namespace.
- Then ingress policies whitelist all namespaces labeled `status=allowed` meaning that any pod running there is also entitled to make connections.
- Finally, ingress policies allow connections from any pod within the **default** namespace labeled `app=my_app`.
- Egress rules are totally permissive, allowing all outgoing connections.

Create the new network policy running the following command:

```command
[environment local]
kubectl create -f ~/kube-security/allow-test.yaml
```

Let's repeat the example above to check what is different:

1. First, according to the new policy, labeling **dmz** namespace would be enough for including all pods inside it to the whitelist, use the following command for doing so:

    ```command
[environment local]
kubectl label namespace dmz status=allowed
    ```

2. Create a new interactive session in the **default** namespace with its corresponding label marking the container as isolated running the following command:

    ```command
[environment local]
kubectl run network-test --rm -ti --restart=Never --namespace=default --labels="status=isolated" --image=busybox /bin/sh
    ```

3. From the new session perform a ping over `dmz-test` container:

    ```super_user
[environment third]
ping <^>dmz-test_IP_address<^>
    ```

4. The ping will succeed this time, because the new network policy has no egress restrictions. Stop the ping command pressing **Ctl+C** and take note of `network-test` IP address:

    ```super_user
[environment third]
ip a
    ```

5. And now from the open `dmz-test` session try to ping the isolated container:

    ```super_user
[environment fourth]
ping <^>network-test_IP_address<^>
    ```

The request will succeed as expected because now the **dmz** namespace and all its pods have ingress permissions. Cancel the ping request pressing **Ctrl+C** and then close all disposable containers running sessions typing **exit** and pressing **ENTER**.

Using network policies you are implementing an additional layer of protection. Depending on your deployment requirements you can define as many rules as needed. A key benefit of this approach is that you can start denying all network access and then open your pods connections only to the necessary services/pods.

Summing up,  security hardening was focused so far on:

- Enforcing RBAC for all users **and** services within the cluster.
- Using authorization roles for fine-grained control over users and services.
- Enhancing network security by means of network access policies.

In the coming sections you will learn how adding Admission Controllers can improve the security of the cluster by complementing the authentication and authorization stages.

## Step Nine - Kubernetes Admission Controllers

It has been explained several times that each Kubernetes request goes through a sequential process of authentication and authorization. That is not optional, even if you configure an insecure **anonymous** user the authentication and authorization occur with each call. On the other hand, [Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) are optional (but strongly recommended) plug-ins that are compiled into the kube-apiserver binary with the objective to broaden security options. 

![Admission Control](https://i.imgur.com/P6PlgiE.png)

Admission controllers intercept requests after they pass the authentication and authorization and then executes its code just before the object persistence. While the outcome of either authentication or authorization check is a boolean (allow or deny the request) admission controllers are more diverse, they can validate requests in the same manner but also some admission controllers can **mutate** the requests and modify the objects they admit. 

Is beyond the scope of this guide explaining admission controllers theory but from a cluster security perspective, it's important to understand that:

- Admission controllers can be activated or deactivated editing master node's `/etc/kubernetes/manifests/kube-apiserver.yaml` searching the corresponding `enable-admission-plugins` or `disable-admission-plugins` flag. The different plug-ins are comma separated.
- When using `kubeadm` for cluster bootstrapping several admission controllers are activated by default. It's worth mentioning that those admission controllers are not always shown in the `kube-apiserver.yaml` manifest. A list of [what is activated by default](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubeapiserver/options/plugins.go#L130-L140) is included in Kubernetes code.
- If any of the controllers reject the request then the **entire request is denied immediately** and an error is returned to the user. This is one reason why is important to completely understand each admission controller before activating it.
- Although admission controllers are optional, some are necessary as stated in the Kubernetes official documentation: *Many advanced features in Kubernetes require an admission controller to be enabled in order to properly support the feature*. This reinforces the prior point, before disabling an admission controller be sure to understand its consequences.

The next sections will cover the most common admission controllers: `ResourceQuota`, `LimitRanger` and `PodSecurityPolicy` and their relevance as a way to enhance cluster security.

### Namespace Resource Quotas

In a previous section, you learned how authorization module lets you restrict users to a specific namespace what is useful to separate environments. This section extends the security benefits of namespaces by means of using the **ResourceQuota** admission controller.

Defining **ResourceQuota** objects allows Kubernetes administrators restricting:

* **Compute resources:** CPU and memory can be limited to a certain value within the namespace. Once set, a resource pre-check is run to ensure that limits are not exceeded during pod creation. If you share one cluster for production and development this could be helpful to avoid resource starvation on production and let you be more permissive in certain namespaces.
* **Storage resources:** similar to CPU and memory you can limit namespace storage consumption. Restricting persistent volumes storage resources is a good practice not only to enhance security but also I/O performance.
* **Objects count:** you can also limit any standard namespace resource type like persistent volume claims, services, secrets, configmaps, pods, replication controllers, load balancers and even resource quotas within the namespace. This offers you a great flexibility, because you may want to restrict certain namespace objects, like pods, for example, to prevent a malicious program to deplete your cluster resources creating many pods in the cluster.

This admission controller is enabled by default when using `kubeadm` during the cluster bootstrapping, so all you need to implement this feature is start creating the definitions.

Let's implement this controller in the **default** namespace. The first step is saving the object file:

```command
[environment local]
nano ~/kube-security/resource-quota-default.yaml
```

And pasting the following definition:

```
[label ~/kube-security/resource-quota-default.yaml]
apiVersion: v1
kind: ResourceQuota
metadata:
  name: resource-quota-default
spec:
  hard:
    pods: "1"
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    configmaps: "5"
    persistentvolumeclaims: "1"
    replicationcontrollers: "10"
    secrets: "3"
    services: "4"
    services.loadbalancers: "1"
```

Now, create the object in the namespace running the usual command:

```command
[environment local]
kubectl create -f ~/kube-security/resource-quota-default.yaml --namespace=default
```

That's it. You can check the **default** namespace quotas issuing the command `describe quota`:

```command
[environment local]
kubectl describe quota --namespace=default
```

And you will get an output like this:

```
[secondary_label Output]
Name:                   resource-quota-default
Namespace:              default
Resource                Used  Hard
--------                ----  ----
configmaps              0     5
limits.cpu              0     2
limits.memory           0     2Gi
persistentvolumeclaims  0     1
pods                    0     1
replicationcontrollers  0     10
requests.cpu            0     1
requests.memory         0     1Gi
secrets                 0     3
services                0     4
services.loadbalancers  0     1
```

<$>[note]
**Note:** resources quotas are expressed in absolute units, take this into account because adding more nodes won't automatically assign more proportional resources to namespaces.
<$>

You may notice that the example above includes two specifications for CPU and memory resources: **limits** and **requests**. Without diving too much into Kubernetes theory, limits are **hard caps** for resources while requests are more a **soft limit** for them. A container exceeding their limit may be terminated if another container needs the resources. That explains why the limits are normally higher than the requests.

Let's demonstrate how this works creating a custom pod using a simple `busybox` image, first create the file:

```command
[environment local]
nano ~/kube-security/quota-test.yaml
```

And then copy and paste the following definition:

```
[label ~/kube-security/quota-test.yaml]
apiVersion: v1
kind: Pod
metadata:
  name: quota-test
spec:
  containers:
  - name: quota-test
    image: busybox
    command: [ "/bin/sh", "-c", "--" ]
    args: [ "while true; do sleep 30; done;" ]
    resources:
      limits:
        memory: "512Mi"
        cpu: "500m" 
      requests:
        memory: "256Mi"
        cpu: "300m"
```

Deploy the pod as usual running the following command:

```command
[environment local]
kubectl create -f ~/kube-security/quota-test.yaml
```

This example is intentionally using a simple command that keeps the container running until is manually terminated. Check again the namespace quotas using the same command than before:

```command
[environment local]
kubectl describe quota --namespace=default
```

You will get an output very similar to this one:

```
[secondary_label Output]
Name:                   resource-quota-default
Namespace:              default
Resource                Used   Hard
--------                ----   ----
configmaps              0      5
limits.cpu              <^>500m<^>   2
limits.memory           <^>512Mi<^>  2Gi
persistentvolumeclaims  0      1
pods                    <^>1<^>      1
replicationcontrollers  0      10
requests.cpu            <^>300m<^>   1
requests.memory         <^>256Mi<^>  1Gi
secrets                 <^>1<^>      3
services                0      4
services.loadbalancers  0      1
```

As you can see, everything is working as expected. You now can try creating a new pod without specifying quotas:

```command
[environment local]
kubectl run nginx --image=nginx --port=80 --restart=Never
```

That will throw you an error similar to the following:

```
[secondary_label Output]
Error from server (Forbidden): pods "nginx" is forbidden: failed quota: resources-quota: <^>must specify limits.cpu,limits.memory,requests.cpu,requests.memory<^>
```

And finally, what would happen if you try to deploy a new pod specifying quotas but knowing that will exceed the pod limit set? Create another pod file:

```command
[environment local]
nano ~/kube-security/another-pod.yaml
```

Copy and paste the following definition:

```
[label ~/kube-security/another-pod.yaml]
apiVersion: v1
kind: Pod
metadata:
  name: another-pod
spec:
  containers:
  - name: another-pod
    image: busybox
    command: [ "/bin/sh", "-c", "--" ]
    args: [ "while true; do sleep 30; done;" ]
    resources:
      limits:
        memory: "512Mi"
        cpu: "500m" 
      requests:
        memory: "256Mi"
        cpu: "300m"
```

And create the new pod as usual with the command:

```command
[environment local]
kubectl create -f ~/kube-security/another-pod.yaml
```

You will see an error similar to this one:

```
[secondary_label Output]
Error from server (Forbidden): error when creating "/home/<^>linux-user<^>/kube-security/another-pod.yaml": pods "another-pod" is forbidden: <^>exceeded quota:<^> resource-quota-default, <^>requested: pods=1, used: pods=1, limited: pods=1<^>
```

It's time to check the admission controllers theory, listing the pods using the command:

```command
[environment local]
kubectl get pods
```

You will see an output like this one:

```
[secondary_label Output]
NAME         READY     STATUS    RESTARTS   AGE
quota-test   1/1       Running   0          21m
```

As expected, no additional pods where created because admission controllers act before the object persistence. 

One final word about namespace quotas. As stated at the beginning of the section, using resource quotas is not only applicable to pods, CPU, and memory but also: storage and objects giving you a huge flexibility to control your application and possible escalation attacks. That's why the responsibility of setting quotas is by default exclusive of users with the `cluster-admin` rights, like your current user **kubernetes-admin**. This prevents other users from changing quotas and bypass restrictions. However, normal users can still verify namespace quotas:

```command
[environment local]
kubectl auth can-i get quota --namespace default --user=sammy
```

As you know by now, you could grant this permission to any user you want creating the corresponding custom role and role binding, but if you are planning doing so just be aware of the security implications involved. For more information about resource quotas please read the [official documentation.](https://kubernetes.io/docs/concepts/policy/resource-quotas/)

### Container Resources Limitation 

In the previous section the `ResourceQuota` admission controller, enabled namespace-wide resources restrictions. But what happens if one (malicious) container consumes all available namespace resources **without** exceeding the established quota? With the `LimitRanger` admission controller preventing that kind of scenario is very easy. This controller enforces the limits declared in the `LimitRange` object in a similar way how `ResourceQuota` does, being the difference its scope: `LimitRange` validates and **mutates** containers instead of namespaces.

To fully understand how it works, create the object file:

```command
[environment local]
nano ~/kube-security/limit-ranges-default.yaml
```

And paste the following declaration on it:

```
[label ~/kube-security/limit-ranges-default.yaml]
apiVersion: v1  
kind: LimitRange  
metadata:  
  name: limit-ranges-default  
spec:  
  limits:  
  - default:  
      memory: 512Mi  
      cpu: "500m"  
    defaultRequest:  
      memory: 256Mi  
      cpu: "300m"  
    type: Container
```

Deploy it to the API server using the usual command:

```command
[environment local]
kubectl create -f ~/kube-security/limit-ranges-default.yaml --namespace=default
```

You can list this object running the following command:

```command
[environment local]
kubectl get limitrange
```

Additionally, you can now check the namespace assigned limits running the following command:

```command
[environment local]
kubectl describe limits --namespace=default
```

The output will be similar to the shown below:

```
[secondary_label Output]
NAME         READY     STATUS    RESTARTS   AGE
quota-test   1/1       Running   0          21m
```

Before continuing, change the **default** namespace pod limit (currently one). Do it modifying the `ResourceQuota` object directly on the API server, for ease editing the object, environmental variable `KUBE_EDITOR` is set to `nano` before passing the command:

```command
[environment local]
KUBE_EDITOR="nano" kubectl edit resourcequota/resource-quota-default
```

Under the **spec** section edit the **pods** limit and set it to three, then save and close the file. Check the new changes listing the namespace quotas using the command:

```command
[environment local]
kubectl describe quota
```

The output should be similar to the following:

```
[secondary_label Output]
Name:                   resource-quota-default
Namespace:              default
Resource                Used   Hard
--------                ----   ----
configmaps              0      5
limits.cpu              500m   2
limits.memory           512Mi  2Gi
persistentvolumeclaims  0      1
pods                    1      <^>3<^>
replicationcontrollers  0      10
requests.cpu            300m   1
requests.memory         256Mi  1Gi
secrets                 1      3
services                0      4
services.loadbalancers  0      1
```

Now that everything is ready, deploy the `nginx` pod that failed in the last section using the command:

```command
[environment local]
kubectl run nginx --image=nginx --port=80 --restart=Never
```

This time you won't get any error. List the namespace quotas again to verify how resources are being consumed:

```command
[environment local]
kubectl describe quota
```

The output will be similar to this one:

```
[secondary_label Output]
Name:                   resource-quota-default
Namespace:              default
Resource                Used   Hard
--------                ----   ----
configmaps              0      5
limits.cpu              <^>1<^>      2
limits.memory           <^>1Gi<^>    2Gi
persistentvolumeclaims  0      1
pods                    <^>2<^>      3
replicationcontrollers  0      10
requests.cpu            <^>600m<^>   1
requests.memory         <^>512Mi<^>  1Gi
secrets                 1      3
services                <^>1<^>      4
services.loadbalancers  0      1
```

The `LimitRanger` admission controller **mutated** the initial request and used the default limits specified in the object definition. You can corroborate the mutation running the following command:

```command
[environment local]
kubectl get pod nginx -o yaml
```

The container specification section should look similar to the following one:

```
[secondary_label Output]
spec:
  containers:
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: nginx
    ports:
    - containerPort: 80
      protocol: TCP
    <^>resources:<^>
      <^>limits:<^>
        <^>cpu: 500m<^>
        <^>memory: 512Mi<^>
      <^>requests:<^>
        <^>cpu: 300m<^>
        <^>memory: 256Mi<^>
```

This would be the same result as if you manually declared the **resources** and **requests** into the container specifications. 

As you can see, using together  `ResourceQuota` and `LimitRanger` admission controllers you can achieve great control over the cluster resources usage. In this guide, only one resource quota was created but can create as many as you need. Kubernetes flexibility allows you to use these controllers and create a kind of Quality of Service (QoS) template assigning resources depending on the priority of your applications. 

### Pod Security Contexts and Pod Security Policies

Little by little, the environment surrounding pods and their respective containers is getting more secure through using admission controllers. This section will introduce a controller affecting pods security more directly: `PodSecurityPolicy`.

Let's start differentiating a pod security **context** from a pod security **policy**. A good simplification would be looking at contexts as *security-oriented templates* for pods. In other words, they control security parameters for pods and hence they affect all containers and volumes within them. You can decide including or not these specifications, actually, you already created several pods without them.

On the other hand, Pod Security Policies (PSP) are referencing the admission controller `PodSecurityPolicy`.  As you might remember, admission controllers intercept all requests sent to the API server and execute their code before object persistence, meaning that any call failing their verification will be rejected. Therefore, once enabled pod security policies will validate all pods, not only a particular set that you decide to. This particular admission controller determines if the request is valid based on:

* **Current Pod Security Policies:** PSP are read in alphabetical order and the first that successfully validates all conditions is used. In other words, you can use PSP as a way to enforce a minimal set of rules for the pod security context.
* **Role Based Authorization:** the controller checks if the user/service has authorization for using the validated policy. This means that you can create custom-tailored PSP for certain users, services, and applications.

Using pod security contexts you can define several aspects for your pods:

- Permissions to access objects based on user and group IDs. In the case of files, this is the equivalent of using Linux standard filesystem permissions.
- SELinux or AppArmor profiles settings.
- Seccomp filters for process's system calls.
- Privilege escalation settings (applicable to containers) allowing you to decide if a container can gain `root` permissions (privileged).

In addition to the parameters specified above, while using pod security policies you can enforce:

* **Host namespaces:** determines if the pods can access/share several host's resources like processes IDs (HostPID), host IPC namespace (HostIPC), host's loopback device and network (HostNetwork) and host's ports (HostPorts) 
* **Volumes and file systems:** gives you control over what volume types (Volumes) and host's filesystem paths (AllowedHostPaths) the containers can access allowing you to whitelist volumes and paths that you consider secure. You can also implement the same `fsGroup` directive used in pods security context for accessing file system based on the group ID and even further, you can use the `ReadOnlyRootFilesystem` to force containers to run in a no writable filesystem. 
* **FlexVolume Drivers:** specifies a whitelist of allowed flex volume drivers.
* **Users and Groups:** most containers use **root** as its default user. For production environments that behavior represents a security risk. You can limit the user (or group) using `RunAsUser`, `MustRunAs`, `MustRunAsNonRoot` or `RunAsAny` specifications. 

Let's create two pods to fully understand the differences between pod security contexts and pod security policies.

### Pod Security Contexts

Before diving into the topic switch to **dmz** namespace as **sammy** user:

```command
[environment local]
kubectl config use-context sammy-dmz
```

Now save a new pod definition including pod security context specifications: 

```command
[environment local]
nano ~/kube-security/pod-security-context-test.yaml
```

Copy and paste the following content:

```
[label ~/kube-security/pod-security-context-test.yaml]
apiVersion: v1
kind: Pod
metadata:
  name: pod-security-context-test
spec:
  securityContext:
    runAsUser: 1500
  containers:
  - name: security-context-test
    image: busybox
    command: [ "/bin/sh", "-c", "--" ]
    args: [ "while true; do sleep 30; done;" ]
    securityContext:
      allowPrivilegeEscalation: false
```

Create the pod as usual:

```command
[environment local]
kubectl create -f ~/kube-security/pod-security-context-test.yaml
```

The example pod only includes two basic `securityContext` definitions that force the container using a non-root user, in this case, UID: 1500. You can confirm it running:

```command
[environment local]
kubectl exec pod-security-context-test -- id
```

The output will be similar to the following:

```
[secondary_label Output]
uid=<^>1500<^> gid=0(root)
```

As you can tell, the container is not using the **root** user. Moreover, the specific argument `allowPrivilegeEscalation: false` denies any possibility to become **root** in this container. 

You can now delete the example pod to keep the namespace clean:

```command
[environment local]
kubectl delete pod pod-security-context-test
```

This is just an illustrative example of what pod security contexts can do. Without much effort, you can enforce using non-privileged users. Since they are part of pod specifications any user with `edit` authorization, like **sammy**, can utilize them without any obstacle. But its flexibility could become a weakness too because is up to users the decision of including the security context or not.  Which leads again to pod security policies and its global scope.

### Pod Security Policies (PSP)

Using pod security contexts was as easy as including few additional specifications in the pod definition file. Implementing pod security policies is not difficult, however, a pre-requisite is enabling the controller in the API server.

<$>[warning]
**Warning:** a word of caution before using pod security policies. There is a good reason why this admission controller is not been activated by default and that is because it controls security sensitive aspects of the pod specification **globally**. Once active, pods creation will be denied unless pod security policies are created and authorized. This applies to **all namespaces** including `kube-system`.
<$>

Let's start creating an absolutely restrictive pod security policy taken from Kubernetes documentation:

```command
[environment local]
nano ~/kube-security/restrictive-psp.yaml
```

Copy and paste into the file the following content:

```
[label restrictive-psp.yaml]
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restrictive-psp
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'docker/default'
    apparmor.security.beta.kubernetes.io/allowedProfileNames: 'runtime/default'
    seccomp.security.alpha.kubernetes.io/defaultProfileName:  'docker/default'
    apparmor.security.beta.kubernetes.io/defaultProfileName:  'runtime/default'
spec:
  privileged: false
  # Required to prevent escalations to root.
  allowPrivilegeEscalation: false
  # This is redundant with non-root + disallow privilege escalation,
  # but we can provide it for defense in depth.
  requiredDropCapabilities:
    - ALL
  # Allow core volume types.
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    # Assume that persistentVolumes set up by the cluster admin are safe to use.
    - 'persistentVolumeClaim'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    # Require the container to run without root privileges.
    rule: 'MustRunAsNonRoot'
  seLinux:
    # This policy assumes the nodes are using AppArmor rather than SELinux.
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
      # Forbid adding the root group.
      - min: 1
        max: 65535
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      # Forbid adding the root group.
      - min: 1
        max: 65535
  readOnlyRootFilesystem: false
```

That is a lot of information to digest:

- In the annotations section, some `seccomp` and `apparmor` directives are applied to enhance the container isolation from the host operating system.
- Then on the specifications section, it begins forbidding escalations to root, even redundantly.
- Continues allowing only the necessary volumes types, meaning the rest will be denied.
- After that, several host resources are protected, including network, host's IPC and using host PIDs.
- Once again, the policy prohibits using the **root** user in a very explicit way, not only with the `MustRunAsNonRoot` rule but also restricting user and group ranges.

Switch back to **kubernetes-admin** superuser and create the policy:

```command
[environment local]
kubectl config use-context kubeadmin-dmz
kubectl create -f restrictive-psp.yaml
```

Now it's time to activate `PodSecurityPolicy` admission controller in the API server, for doing so, you will need starting a SSH session in your Master node as usual:

```command
[environment second]
ssh ubuntu@<^>master_ip<^>
```

Once there, edit the `kube-apiserver` main file running the following command:

```command
[environment second]
sudo nano /etc/kubernetes/manifests/kube-apiserver.yaml
```

Look for the `--enable-admission-plugins` flag and add the new controller (comma separated) as shown below:

```
[secondary_label Output]
- --enable-admission-plugins=NodeRestriction,<^>PodSecurityPolicy<^>
```

Save and close the file, the admission controller will start working immediately.

Close your remote SSH session. You can now continue using your local client `kubectl`. Try creating a new persistent pod using `nginx` standard image running the following command:

```command
[environment local]
kubectl run nginx --image=nginx --port=80 --restart=Never
```

You will see the accustomed message saying that the pod was created, but verify it by listing the pods using the following command:

```command
[environment local]
kubectl get pods
```

You will notice it wasn't deployed. An error message indicates there is something wrong in the container configuration:

```
[secondary_label Container Error During Creation]
NAME                   READY     STATUS                       RESTARTS   AGE
nginx                  0/1       <^>CreateContainerConfigError<^>   0          4s
```

Dig into the pod description to visualize the error using the following command:

```command
[environment local]
kubectl describe pod nginx
```

At the end of the description you will find the information regarding the reason for the container failure: 

```
[secondary_label Container Error During Creation]
Error: container has <^>runAsNonRoot<^> and image will run as <^>root<^>
```

The error is expected because you are enforcing non-root users in the policy. But why the pod was accepted in the first place? Admission controllers are supposed to act before the object persistence what happened then?

The best way to understand this behavior is creating another pod but this time with the **adm** user:

```command
[environment local]
kubectl config use-context adm-dmz
kubectl run nginx-adm --image=nginx --port=80 --restart=Never
```

This time you will see an error similar to this one:

```
[secondary_label Output]
Error from server (Forbidden): pods "nginx-adm" is forbidden: <^>unable to validate against any pod security policy<^>: []
```

Listing pods will confirm the object was indeed rejected, there is no `nginx-adm` pod. A similar result wil be obtained with the normal user **sammy**. The main reason behind why this is happening is related to how PSP and RBAC work together. As explained earlier, pod security policies first try to validate policies and then look for authorized users. The controller found `restrictive-psp` policy but neither **adm** or **sammy** have authorization to **use** the PSP. On the other hand, **kubernetes-admin** is a superuser, and as such has no authorization limits, **but** the `nginx` pod failed as well because it didn't comply with the non-root policy.

Long story short, you need meeting two conditions in order to successfully using pod security policies:

- You need at least one PSP already created, only superusers can create these policies.
- You need authorization for using the policy.

The combination of the two conditions offers you the ability to enforce custom tailored security contexts for your applications or namespaces. Let's say that you want to overcome all security restrictions in the **dmz** namespace. One way to achieve that goal is creating another pod security policy using the following command:

```command
[environment local]
nano ~/kube-security/allow-all-psp.yaml
```

Copy and paste the following content into the file:

```
[label allow-all-psp.yaml]
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: allow-all-psp
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: '*'
spec:
  privileged: true
  allowPrivilegeEscalation: true
  allowedCapabilities:
  - '*'
  volumes:
  - '*'
  hostNetwork: true
  hostPorts:
  - min: 0
    max: 65535
  hostIPC: true
  hostPID: true
  runAsUser:
    rule: 'RunAsAny'
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```

If you compare this policy with the previous one, you will find that is the opposite in every aspect, consequently is named `allow-all-psp` because is like having no pod security at all.

Change again to the superuser context and create the new policy like before:

```command
[environment local]
kubectl config use-context kubeadmin-dmz
kubectl create -f allow-all-psp.yaml
```

From this point, you only need applying what you learned in the RBAC authorization section, first create a cluster role allowing the **use** verb for the PSP resource:

```command
[environment local]
nano ~/kube-security/allow-all-psp-clusterrole.yaml
```

Copy and paste the following declaration:

```
[label allow-all-psp-clusterrole.yaml]
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: allow-all-psp-clusterrole
rules:
- apiGroups: ['policy']
  resources: ['podsecuritypolicies']
  verbs:     ['use']
  resourceNames:
  - allow-all-psp
```

And then create a role binding to all authenticated users in the **dmz** namespace.

```command
[environment local]
nano ~/kube-security/allow-all-psp-rolebinding.yaml
```

Copy and paste the following declaration:

```
[label allow-all-psp-rolebinding.yaml]
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: allow-all-psp-rolebinding
  namespace: dmz
roleRef:
  kind: ClusterRole
  name: allow-all-psp-clusterrole
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: Group
  apiGroup: rbac.authorization.k8s.io
  name: system:authenticated
```

Create the new objects running as usual:

```command
[environment local]
kubectl create -f allow-all-psp-clusterrole.yaml 
kubectl create -f allow-all-psp-rolebinding.yaml
```

In order to test the new policy switch to **sammy** user working on **dmz** namespace using this command:

```command
[environment local]
kubectl use-context sammy-dmz
```

Now try creating a pod:

```command
[environment local]
kubectl run nginx-sammy --image=nginx --port=80 --restart=Never 
```

And list pods to confirm everything is working as expected:

```command
[environment local]
kubectl get pods
```

The pod was created without any error using the least permissive user because this namespace is now entitled to accept any security context (or no context at all).

Clean-up the namespace running the commands:

```command
[environment local]
kubectl delete pod nginx-sammy
kubectl delete pod nginx
```

As you can tell, the last few sections improved security within namespaces and pods implementing:

- Resources Limits for namespaces and containers.  
- Security policies (templates) for pods affecting the containers and volumes inside them.

As far as Kubernetes is concerned, using PSP is the closest it can get in terms of improving the security of the application. The next chapter will introduce the basic concepts to consider when designing containerized applications.

## Step Ten - Application Level Security

Container security is a topic that deserves a tutorial on its own. The technology is advancing so fast that the security aspect constitutes a huge challenge. This chapter will provide an overview of the main aspects of container security affecting Kubernetes clusters.

### Applications as Micro-services

Applications are, in the end, the reason for cluster existence. Orchestration technologies arose as a solution for high-availability easy to scale micro-services (applications). Up to this point, this guide has covered many different potential attack vectors targeting infrastructure, nodes, cluster, but very few about applications. 

Micro-services are not exempt from attacks, on the contrary, they pose a high-security risk if they are not properly configured. To illustrate this point, a very simple application will be deployed into the cluster.

1. First, create a new local directory `~/kube-security/myapp` and then change to it:

    ```command
[environment local]
mkdir ~/kube-security/myapp && cd ~/kube-security/myapp
    ```

2. Create the application file `app.sh`: 

    ```command
[environment local]
nano ~/kube-security/myapp/app.sh
    ```

3. Copy and paste the following script on it:

    ```
[label ~/kube-security/myapp/app.sh]
#!/bin/sh
echo "############  MY AWESOME APP  ############"
echo "#                                        #"
echo "#  Pod  hostname: " `hostname`
echo "#  Node hostname: " `cat /tmp/hostname`
echo "#----------------------------------------#"
    ```

4. Now create the `Dockerfile` in the same directory:

    ```command
[environment local]
nano ~/kube-security/myapp/Dockerfile
    ```

5. Copy and paste the following content:

    ```
[label /myapp/Dockerfile]
FROM alpine:latest
RUN mkdir /home/app
ADD ./app.sh /home/app
RUN chmod +x /home/app/app.sh
WORKDIR /home/app
    ```

6. Now. create the Docker image using the following command:

    ```command
[environment local]
docker build -f Dockerfile -t <^>your_dockerhub_username<^>/my_app:standard .
    ```

7. Log into your Docker Hub account and push the image running the following commands:

    ```command
[environment local]
docker login
docker push <^>your_dockerhub_username<^>/my_app:standard
    ```

8. Now you can return to your previous working directory `~/kube-security`:

    ```command
[environment local]
cd ..
    ```

9. Create the application pod definition file using the usual command:

    ```command
[environment local]
nano ~/kube-security/standard-app-pod.yaml
    ```

10. Copy and paste the pod definition shown below:

    ```
[label ~/kube-security/standard-app-pod.yaml]
apiVersion: v1
kind: Pod
metadata:
  name: standard-app-pod
spec:
  containers:
  - name: standard-container
    image: <^>your_dockerhub_username<^>/my_app:standard
    volumeMounts:
    - mountPath: /tmp
      name: test-volume
    command: [ "/bin/sh", "-c", "--" ]
    args: [ "while true; do sleep 30; done;" ]
  volumes:
  - name: test-volume
    hostPath:
      path: /etc
    ```

11. Change to **sammy** user in the **dmz** namespace and create the pod using the following commands:

    ```command
[environment local]
kubectl config use-context sammy-dmz
kubectl create -f standard-app-pod.yaml
    ```

12. Test the application to confirm that everything is working as expected:

    ```command
[environment local]
kubectl exec standard-app-pod -- ./app.sh
    ```

13. The resulting output will be similar to this one:

    ```
[secondary_label Output]
############  MY AWESOME APP  ############
#                                        #
#  Pod hostname :  standard-app-pod
#  Node hostname:  <^>worker<^>
#----------------------------------------#
    ```

As you can see, the script `app.sh` just prints to the console the running pod hostname and its corresponding node hostname. You may be asking where is the point of this example? 

14. Start an interactive console inside the application's container running the following command:

    ```command
[environment local]
kubectl exec -it standard-app-pod -- /bin/sh
    ```

15. Change to the `tmp` directory where the host's `/etc` is mounted through the container volume:

    ```super_user
[environment third]
cd /tmp
    ```

16. Now **edit** the `hostname` file (you can use `vi`) and change its value to `hacked-worker`.

17. Exit the container by either pressing **Ctrl+D** or typing **exit** and pressing **ENTER**.

18. Run again the application and watch the new output:

    ```command
[environment local]
kubectl exec standard-app-pod -- ./app.sh
    ```

The new output should look similar to the following:

```
[secondary_label Output]
############  MY AWESOME APP  ############
#                                        #
#  Container ID :  standard-app-pod
#  Node hostname:  <^>hacked-worker<^>
#----------------------------------------#
```

You just **modified** the node's hostname through the container, in theory, applications inside containers are isolated from the host but that was not the case. Let's analyze how that was possible:

- Standard Docker images (like **alpine**) use **root** as the default user. That is great for development environments because you have access to the entire file system, but for production using **root** should be avoided at all cost.
- The example pod intentionally used a `hostPath` volume. That is a risky practice because you are granting the container direct access to host's file system.

The combination of the above conditions allowed a simple hack. Using the concepts learned in the prior sections you can mitigate the inherent risk of this scenario. If you try creating the same pod in the **default** namespace it will fail for several reasons:

- For starters you need authorizing the user **sammy** for using an appropriate PSP.
- As explained previously, admission controllers can mutate pods meaning that even when no security context is specified the PSP can force the container to run with a non-root user. The problem is that it won't allow a `hostPath` volume and will fail.

Any application with intentions of running in the **default** namespace is expected to:

* **Pass authentication and authorization checks:** in this guide pods where deployed manually by normal users, in practice, each application **should have** a service account bound to it. Proper permissions may be granted depending of application scope.
* **Comply with current Pod Security Policies:** meaning the application should be prepared for execution with non-privileged users, should use safe volume types, and won't try accessing host's resources.
* **Comply with namespace quotas and limits:** all deployments will be validated by the `ResourceQuota` and `LimitRanger` admission controllers against the hard limits you set previously. 
* **Comply with namespace network policies:** as explained in the corresponding section, you can **isolate** complete namespaces or individual pods that meet certain criteria. This allows you to restrict potentially dangerous application even further if necessary.

The coming sections will introduce some additional security best practices to ensure that your container layer is as secure as possible.

### Container Image and Registry Security

Formally speaking, nothing stops you from using a public container registry for storing your images. But from a security standpoint, that's not recommended . Your application image and all base images used to build it are exposed to the Internet which means they are vulnerable to malicious hacks.

Ideally, your production cluster should use a [private container registry](https://www.digitalocean.com/community/tutorials/how-to-build-docker-images-and-host-a-docker-image-repository-with-gitlab) where you can safely store approved base images and of course your main applications images as well. 

You could also go further and enhance security forcing containers to **always** pull images. Implementing this practice is very easy:

- Include in the containers section of your pod specification the `imagePullPolicy: Always`. This is a good practice to force the policy at the container level.
- The default image pull policy is `IfNotPresent`, this saves some bandwidth when the image is already present in the cluster. You can override that setting enabling `AlwaysPullImages` admission controller and forcing image policy to always. 

<$>[note]
**Note:** in order to use private repositories you need additional configuration in your cluster and store authentication **secrets** beforehand. For more information please read the [official Kubernetes documentation](https://kubernetes.io/docs/concepts/containers/images/#using-a-private-registry)
<$>

### Service Accounts

It has been mentioned several times during this guide that applications should have its own service account. Two important arguments supporting this practice are the ability to identify the applications easier (logs, audits, monitoring) and the power of controlling its authorization level in real time. 

Truth is, all pods deployed so far **had**, in fact, a service account assigned to them. Kubernetes automatically creates a **default** service account with its corresponding credentials on each namespace. When creating a pod, if you do not explicitly specify a service account, that account (default) is assigned to it. 

You can confirm the above checking the `standard-app-pod` application using the following command:

```command
[environment local]
kubectl get pods/standard-app-pod -o yaml
```

A long output will appear with descriptive information about the pod. Under the containers specification section you will see the service account information as shown below:

```
[secondary_label Output
serviceAccountName: <^>default<^>
```

If you create more pods in the `dmz` zone they will use the same service account, try it deploying a new pod using the following command:

```command
[environment local]
kubectl run nginx --image=nginx --port=80 --restart=Never
```

Get the details in a similar way than before running the following command:

```command
[environment local]
kubectl get pods/nginx -o yaml
```

Both share the same `default` service account. You can also corroborate that each namespace has its own `default` service account running:

```command
[environment local]
kubectl config use-context kubeadmin-dmz 
kubectl get sa --all-namespaces
```

The first portion of the output will be similar to:

```
[secondary_label Output]
NAMESPACE     NAME                                 SECRETS   AGE
default       <^>default<^>                              1         8d
dmz           <^>default<^>                              1         8d
kube-public   <^>default<^>                              1         8d
```

Not all application are created equal, from a security perspective is not optimal using the same service account for all processes. Create a new SA called `standard-app-pod` using the following command:

```command
[environment local]
kubectl create sa standard-app-sa
```

Kubernetes automatically bound a secret to the SA so its ready for use. Let's test it.

First delete the old `standard-app-pod` application:

```command
[environment local]
kubectl delete pod standard-app-pod
```

Now create a new pod file including the recently added service account:

```command
[environment local]
nano ~/kube-security/standard-app-pod-sa.yaml
```

Copy and paste the following content into the file:

```
[label ~/kube-security/standard-app-pod-sa.yaml]
apiVersion: v1
kind: Pod
metadata:
  name: standard-app-pod-sa
spec:
  serviceAccountName: standard-app-sa
  containers:
  - name: standard-container
    image: <^>your_dockerhub_username<^>/my_app:standard
    volumeMounts:
    - mountPath: /tmp
      name: test-volume
    command: [ "/bin/sh", "-c", "--" ]
    args: [ "while true; do sleep 30; done;" ]
  volumes:
  - name: test-volume
    hostPath:
      path: /etc
```

Create the pod as usual using the commandl:

```command
[environment local]
kubectl create -f standard-app-pod-sa.yaml
```

And now inspect the pod to confirm that is using the new service account:

```command
[environment local]
kubectl get pods/standard-app-pod-sa -o yaml
```

You will see an output similar to the following:

```
[secondary_label standard-app-pod-sa]
serviceAccount: <^>standard-app-sa<^>
serviceAccountName: <^>standard-app-sa<^>
```

One key benefit of this approach is flexibility, you could assign separate service accounts to each micro-service or you could group some applications using the same SA. Then you can control applications access to resources using RBAC authorization. You can even generate your own credentials for service accounts if you want/need to. From a security perspective, possibilities are endless. Using service accounts is the recommended practice for authorization control over your applications.

## Conclusion

Throughout this guide, you have learned what can be considered as a Kubernetes end-to-end basic security template. Each chapter covered a different angle respecting possible security breaches:

- Risks inherent to any infrastructure facing the Internet.
- The challenge associated with a properly secured authentication mechanism.
- The recommended best practices regarding intra-cluster deployments and security policies.
- Applications as the window to outside world and thus all the added hacking opportunities that must be prevented.

Combining all the suggestions covered in this article you will have a solid foundation for a production Kubernetes cluster deployment, from there you can start hardening individual aspects depending on your scenario.
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE3MDE2MjkyNTMsLTEzODYwODY5NzEsLT
MwNzQ0NTA1OCwxOTM2MjMzMTkzLDcyMTQyMTc1MSwtNDMyODc5
MjQ3LDEyMjg0MDE0NjQsMTM3NzI0MzU2MiwyMDI4NjMyMzk4LC
04MTQxMDc5NDAsNTEwMDkwOTcyLC0yNjMxNjc5MTUsLTcxNzc5
NTM0OSwxOTU4NzU5MDgwLC03MTc3OTUzNDksMTEyNTEzMDk5OS
wxNTEzNjQ3MjIsMjgzMDA3NTUwLDU5OTkyNzQwNCwxMzYyMzg0
MTUwXX0=
-->