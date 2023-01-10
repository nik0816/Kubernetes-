
# Kubernetes cluster creation using kubeadm 

In this section we are going to create an kubernetes single master and worker node cluster using kubeadm. We are going to use amazon ec2 as our virtual machine.So we are going to use free tier instances for this demo or else you can use any sort of instance of your choice.



## PREREQUISITS

### Reference
 - [AWS Account Creation](https://aws.amazon.com/free/?trk=14a4002d-4936-4343-8211-b5a150ca592b&sc_channel=ps&s_kwcid=AL!4422!3!453325184782!e!!g!!aws&ef_id=CjwKCAiAk--dBhABEiwAchIwkYoZQLRQS2ZRfnWP04hWF1VykNnEb-owxBjaV5GJ-pZM4daszc9mHRoCobIQAvD_BwE:G:s&s_kwcid=AL!4422!3!453325184782!e!!g!!aws&all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc&awsf.Free%20Tier%20Types=*all&awsf.Free%20Tier%20Categories=*all)
 - [KUBERNETES](https://kubernetes.io/)
 - [Kubernetes Architecture](https://kubernetes.io/docs/concepts/overview/)
 - [Docker](https://docs.docker.com/get-started/) 
 - [kubernetes ports](https://kubernetes.io/docs/reference/networking/ports-and-protocols/)
 
  ## Ec2 instance configuration

  Ubuntu 20.04 with certain ports are open on your machines.  [See here](https://kubernetes.io/docs/reference/networking/ports-and-protocols/) for more details.
  and 8 gb storage(ebs) is sufficient. 
  
- ### Instance type  
- For master node : t2 medium
- For worker node : t2 micro 

# Installation

We need the super user permission to perform some task so either use root login or add sudo permission before commands to give the super user permission.

## Steps to be performed on both the machines 

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
Step 12: To change the hostname of the nodes you can use the following command 
                   
```bash
  sudo hostnamectl set-hostname (your hostname)                             
``` 

## On Master Node Only	
Step 1:Initialise kubernetes on master node 
                   
```bash
sudo kubeadm init                                
```
Step 2 To make kubectl work for your non-root user, run these commands, which are also part of the kubeadm init output:
                   
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config                                
```
 - Alternatively, if you are the root user, you can run:
```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
``` 
Step 3 Execute the join command on worker node that was previously given to you by the kubeadm init output on the master node. It should look something like this:
                   
```bash
sudo kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866
```
- This token is used to join the worker node with the master node 

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
kubectl get pods 
```
for checking the pods in all namespaces use the following command

```bash
kubectl get pods --all-namespaces
```
