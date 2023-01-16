

# Highly available Stacked Kubernetes cluster creation using kubeadm version before 1.26.0

To setup Highly available multimaster kubernetes cluster using kubeadm on the ubuntu 20.04 follow this document.

For this setup we are going to use aws ec2 instances.

With stacked control plane nodes. This approach requires less infrastructure. The etcd members and control plane nodes are co-located.For reading more about this [see here](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/#stacked-etcd-topology) 

The following documentation will help you with creating three master, one worker node and loadbalancer node using HAproxy.


## PREREQUISITS

### Reference
 - [AWS Account Creation](https://aws.amazon.com/free/?trk=14a4002d-4936-4343-8211-b5a150ca592b&sc_channel=ps&s_kwcid=AL!4422!3!453325184782!e!!g!!aws&ef_id=CjwKCAiAk--dBhABEiwAchIwkYoZQLRQS2ZRfnWP04hWF1VykNnEb-owxBjaV5GJ-pZM4daszc9mHRoCobIQAvD_BwE:G:s&s_kwcid=AL!4422!3!453325184782!e!!g!!aws&all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc&awsf.Free%20Tier%20Types=*all&awsf.Free%20Tier%20Categories=*all)
 - [KUBERNETES](https://kubernetes.io/)
 - [Kubernetes Architecture](https://kubernetes.io/docs/concepts/overview/components/)
 - [Docker](https://docs.docker.com/get-started/) 
 - [kubernetes ports](https://kubernetes.io/docs/reference/networking/ports-and-protocols/) 
  ## Ec2 instance configuration

  - Ubuntu 20.04 with certain ports are open on your machines.  [See here](https://kubernetes.io/docs/reference/networking/ports-and-protocols/) for more details.
    and 8 gb storage(ebs) is sufficient. 
  - All machines must be accessible on the network. For cloud users - single VPC for all machines
  - ssh access from loadbalancer node to all machines (master & worker).
- ## Instance type  


| ROLE | Type of instance | Operating system |  EBS Storage |
| :---         |     :---:      |          ---: | ---:|
| Loadbalancer  |  T2.micro    |Ubuntu 20.04    | 8gb     |
| master1  |  T2 medium    |Ubuntu 20.04    | 8gb     |
| master2  |  T2 medium    |Ubuntu 20.04    | 8gb     |
| master3  |  T2 medium    |Ubuntu 20.04    | 8gb     |
| worker  |  T2.micro    |Ubuntu 20.04    | 8gb     |


# Installation

We need the sudo preveiledge to perform some task so either use root login or add sudo permission before commands to give the super user permission.

Step 1: To change the hostname of the machines you can use the following command 
                   
```bash
  sudo hostnamectl set-hostname (your hostname)                             
``` 
## Setup the loadbalancer 
For this setup we are going to use HA proxy load balancer or you can use any sort of tcp loadbalancer.
### Why do we need loadbalancer?
 As we are using 3 master nodes over here which means users can connect to either of the three api-servers.loadbalancer is used to loadbalance between three api-serves. 

Step 1. Update the machine 
```bash
  sudo apt-get update                                  
```

Step 2. Install haproxy
```bash
  sudo apt-get install haproxy -y                                 
```
Step 3. Edit haproxy configuration
```bash
  vi /etc/haproxy/haproxy.cfg                                 
```
Step 4. Add the below lines to create a frontend configuration for loadbalancer -
```bash
  frontend fe-apiserver
   bind 0.0.0.0:6443
   mode tcp
   option tcplog
   default_backend be-apiserver                                
```
Step 5. Add below lines after the step 4. to create and backend configuration.
```bash
backend be-apiserver
   mode tcp
   option tcplog
   option tcp-check
   balance roundrobin
   default-server inter 10s downinter 5s rise 3 fall 3 slowstart 60s maxconn 250 maxqueue 256 weight 100

       server master1 x.x.x.x:6443 check
       server master2 x.x.x.x:6443 check
       server master3 x.x.x.x:6443 check
                                  
```
Note. Add master nodes private ips over here at x.x.x.x

Step 6. Restart and Verify haproxy
```bash
systemctl restart haproxy
systemctl status haproxy                                
```
Ensure haproxy is in running status.

Step 7. Run nc command as below -
```bash
nc -v localhost 6443
Connection to localhost 6443 port [tcp/*] succeeded!
```

## Steps to be performed on all master and all worker machines 

Step 1: first we are going to update system 
                   
```bash
  sudo apt-get update                                  
```

Step 2: For intra cluster communication/installation of https packages with safety 
                   
```bash
  sudo apt-get install apt-transport-https                                  
```

Step 3: Install the curl and Docker utility on both the nodes by running the following command as sudo in the Terminal of each node.
                   
```bash
  sudo apt install -y curl docker.io                                 
```

Step 4: verify the installation and also check the version number of Docker 
                   
```bash
  docker --version                                  
```   

Step 5: To check the status of docker utility use following command 
                   
```bash
  sudo systemctl status docker                              
```  

Step 6: If the docker utility is not enable . To enable docker utility use following command 
                   
```bash
  sudo systemctl enable docker                              
```  

Step 7: To get the Kubernetes signing key for authentication purpose into kubernetes xenial repository
                   
```bash
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add                              
```  


Step 8: In order to add the Xenial Kubernetes repository
                   
```bash
 sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"                              
``` 

Step 9: Update the system 
                   
```bash
   apt-get update		                             
``` 
Step 10: Install kubeadm ,kubectl ,kubelet and kubernetes-cni(use this command for the choosing the latest version of the following components) 
                   
```bash
  sudo apt-get install -y kubelet kubeadm kubectl kubernetes-cni                              
``` 

To install particular version of the kubelet, kubeadm, kubectl, kubernetes-cni use the following command 
```bash
  sudo apt-get install -qy kubelet=1.2X.X-00 kubectl=1.2X.X-00 kubeadm=1.2X.X-00                               
``` 

Step 11: Turn off the swapoff memory off (we need to turnoff the swap disable for kubelet to perform properly)
                   
```bash
  sudo swapoff -a                              
``` 

## On Masternode 1 Only	
Step 1.Initialise kubernetes on master1 
                   
```bash
sudo kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" --upload-certs --pod-network-cidr=192.168.0.0/16                                
```
- When --upload-certs is used with kubeadm init, the certificates of the primary control plane are encrypted and uploaded in the kubeadm-certs Secret.

The output looks like this 
```bash
...
You can now join any number of control-plane node by running the following command on each as a root:
    kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv \
        --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 \  
        --control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use kubeadm init phase upload-certs to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:
    kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866                           
``` 

## For each additional control plane node you should:

Step 1. Execute the join command that was previously given to you by the kubeadm init output on the master1 node . It should look something like this:
```bash
sudo kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv \
         --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 \
         --control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07``` 
```
- The --control-plane flag tells kubeadm join to create a new control plane.
- The --certificate-key ... will cause the control plane certificates to be downloaded from the kubeadm-certs Secret in the cluster and be decrypted using the given key.
- This token is used to add master nodes to the cluster.

Step 2. Execute the join command on worker node that was previously given to you by the kubeadm init output on the master node. It should look something like this:
                   
```bash
sudo kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866
```
- This token is used to join the worker node with the master node (Bootstraping the worker node.) 
## Configuring kubeconfig on loadbalancer 
 Now that we have configured the master and the worker nodes, its now time to configure Kubeconfig (.kube) on the loadbalancer node. It is completely up to you if you want to use the loadbalancer node to setup kubeconfig. 
 kubeconfig can also be setup externally on a separate machine which has access to loadbalancer node. For the purpose of this demo we will use loadbalancer node to host kubeconfig and kubectl.
 
 Step 1.Create directory .kube on $HOME of root 
```bash
  mkdir -p $HOME/.kube                             
``` 

step 2.Copy the configuration file from any one master node using following command 
```bash
scp master1:/etc/kubernetes/admin.conf $HOME/.kube/config                             
``` 
step 3. provide appropriate ownership to the copied file
```bash
chown $(id -u):$(id -g) $HOME/.kube/config
``` 
Step 4. Install kubectl binary 
```bash
snap install kubectl --classic  
``` 


 ### Installing a Pod network add-on

 You must deploy a Container Network Interface (CNI) based Pod network add-on so that your Pods can communicate with each other. Cluster DNS (CoreDNS) will not start up before a network is installed.Refer the following link for more details [CNI](https://github.com/containernetworking/cni).As of now we are using calico for the installation. 


```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```
For checking the nodes in the cluster use the following command  
```bash
kubectl get nodes
```
For checking the pods in default namespace use following command 

```bash
kubectl get pods --all-namespaces
```
for checking the pods in all namespaces use the following command

```bash
kubectl get pods --all-namespaces
```     
