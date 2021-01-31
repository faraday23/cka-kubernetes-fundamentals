Install Kubernetes
Log into your nodes. If attending in-person instructor led training the node IP addresses will be provided by the
instructor. You will need to use a .pem or .ppk key for access, depending on if you are using ssh from a terminal or
PuTTY. The instructor will provide this to you.

1. Open a terminal session on your first node. For example, connect via PuTTY or SSH session to the first GCP node. The
user name may be different than the one shown, student. The IP used in the example will be different than the one you
will use.

## ssh into node
`ssh -i LFS458.pem student@35.226.100.87`


//make sure to add firewall rule
Warning: Permanently added '35.226.100.87' (ECDSA) to the list of known hosts.


2. Become root and update and upgrade the system. 

## Become root
`sudo -i`

## update and upgrade the system
`apt-get update && apt-get upgrade -y`


3. Install a text editor like nano, vim, or emacs. Any will do, the labs use a popular option, vim.
## install vim
`apt-get install -y vim`

4. The main choices for a container environment are Docker. The default when building the cluster with kubeadm on Ubuntu.

(a) If using Docker:
`apt-get install -y docker.io`


5. Add a new repo for kubernetes. You could also download a tar file or use code from GitHub. Create the file and add an
entry for the main repo for your distribution. We are using the Ubuntu 18.04 but the kubernetes-xenial repo of the
software, also include the key word main. Note there are four sections to the entry.

## create vim file for kubernetes-=xenial repo
`vim /etc/apt/sources.list.d/kubernetes.list`

`deb http://apt.kubernetes.io/ kubernetes-xenial main`

6. Add a GPG key for the packages. The command spans three lines. You can omit the backslash when you type. The OK
is the expected output, not part of the command.

`curl -s \
https://packages.cloud.google.com/apt/doc/apt-key.gpg \
| apt-key add -`


7. Update with the new repo declared, which will download updated repo information.

## update new repo
`apt-get update`



8. Install the software. There are regular releases, the newest of which can be used by omitting the equal sign and version
information on the command line. Historically new versions have lots of changes and a good chance of a bug or five. As
a result we will hold the software at the recent but stable version we install.

`apt-get install -y \
kubeadm=1.18.1-00 kubelet=1.18.1-00 kubectl=1.18.1-00`

`apt-mark hold kubelet kubeadm kubectl`

9. Deciding which pod network to use for Container Networking Interface (CNI) should take into account the expected
demands on the cluster. 
We will use Calico as a network plugin which will allow us to use Network Policies later in the course. Currently
Calico does not deploy using CNI by default. Newer versions of Calico have included RBAC in the main file. Once
downloaded look for the expected IPV4 range for containers to use in the configuration file.

## download and install calico
`wget https://docs.projectcalico.org/manifests/calico.yaml`

10. Use less to page through the file. Look for the IPV4 pool assigned to the containers. There are many different configuration
settings in this file. Take a moment to view the entire file. The CALICO_IPV4POOL_CIDR must match the value
given to kubeadm init in the following step, whatever the value may be. Avoid conflicts with existing IP ranges of the
instance.

## use less command to page through the file. Look for the IPV4 pool assigned to the containers.
```
less calico.yaml

calico.yaml
```

## The default IPv4 pool to create on startup if none exists. Pod IPs will be chosen from this range. Changing this value after installation will have no effect. This should fall within `--cluster-cidr.`
```
 - name: CALICO_IPV4POOL_CIDR
   value: "192.168.0.0/16"
```

11. Find the IP address of the primary interface of the master server. The example below would be the ens4 interface and
an IP of 10.128.0.3, yours may be different.

## show ip address of primary interface
`ip addr show`

12. Add an local DNS alias for our master server. Edit the /etc/hosts file and add the above IP address and assign a
name k8smaster.

## edit etc/hosts file and add the ip address from previous command
`vim /etc/hosts`

`10.128.0.3 k8smaster #<-- Add this line
127.0.0.1 localhost`

13. Create a configuration file for the cluster. There are many options we could include, but will only set the control plane
endpoint, software version to deploy and podSubnet values. After our cluster is initialized we will view other default
values used. Be sure to use the node alias, not the IP so the network certificates will continue to work when we deploy
a load balancer in a future lab.

## set control plane endpoint
`vim kubeadm-config.yaml`

`kubeadm-config.yaml`

`apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: 1.18.1              #<-- Use the word stable for newest version
controlPlaneEndpoint: "k8smaster:6443" #<-- Use the node alias not the IP
networking:
  podSubnet: 192.168.0.0/16            #<-- Match the IP range from the Calico config file`


14. Initialize the master. Read through the output line by line. Expect the output to change as the software matures. At the
end are configuration directions to run as a non-root user. The token is mentioned as well. This information can be found
later with the kubeadm token list command. The output also directs you to create a pod network to the cluster, which
will be our next step. Pass the network settings Calico has in its configuration file, found in the previous step. Please
note: the output lists several commands which following exercise steps will complete.

## initializes a Kubernetes control-plane node and Upload certificates to kubeadm-certs, Save output for future review
`kubeadm init --config=kubeadm-config.yaml --upload-certs | tee kubeadm-init.out`


## You can now join any number of the control-plane node
running the following command on each as root:
`kubeadm join k8smaster:6443 --token vapzqi.et2p9zbkzk29wwth \
--discovery-token-ca-cert-hash sha256:f62bf97d4fba6876e4c3ff645df3fca969c06169dee3865aab9d0bca8ec9f8cd \
--control-plane --certificate-key 911d41fcada89a18210489afaa036cd8e192b1f122ebb1b79cce1818f642fab8`

Please note that the certificate-key gives access to cluster sensitive
data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If
necessary, you can use

## to reload certs afterward
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.
Then you can join any number of worker nodes by running the following
on each as root:

## join number of workder nodes by running the following
`kubeadm join k8smaster:6443 --token vapzqi.et2p9zbkzk29wwth \
--discovery-token-ca-cert-hash sha256:f62bf97d4fba6876e4c3ff645df3fca969c06169dee3865aab9d0bca8ec9f8cd`

15. As suggested in the directions at the end of the previous output we will allow a non-root user admin level access to the
cluster. Take a quick look at the configuration file once it has been copied and the permissions fixed.

## exit root user
`exit`

## make directory
`mkdir -p $HOME/.kube`

## copy files new directory
`sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`

## allows you to change the user and/or group ownership of a given file, directory, or symbolic link.
`sudo chown $(id -u):$(id -g) $HOME/.kube/config`

## Less is a command line utility that displays the contents of a file or a command output, one page at a time. 
```
less .kube/config

apiVersion: v1
clusters:
- cluster:
```

16. Apply the network plugin configuration to your cluster. 
Remember to copy the file to the current, non-root user directory

## copy file to non-root user directory
`sudo cp /root/calico.yaml .`

## apply network plugin
`kubectl apply -f calico.yaml`


17. While many objects have short names, a kubectl command can be a lot to type. We will enable bash auto-completion.
Begin by adding the settings to the current shell. Then update the /.bashrc file to make it persistent. Ensure the
bash-completion package is installed. If it was not installed, log out then back in for the shell completion to work.

## instal bash-completion package
`sudo apt-get install bash-completion -y
<exit and log back in>`

## testing kubectl compeletion
`source <(kubectl completion bash)`

## update .bashrc file to make it persistent
`echo "source <(kubectl completion bash)" >> ˜/.bashrc`

18. Test by describing the node again. Type the first three letters of the sub-command then type the Tab key. Auto-completion
assumes the default namespace. Pass the namespace first to use auto-completion with a different namespace. By
pressing Tab multiple times you will see a list of possible values. Continue typing until a unique name is used. First look
at the current node (your node name may not start with lfs458-), then look at pods in the kube-system namespace. If
you see an error instead such as -bash: _get_comp_words_by_ref: command not found revisit the previous step,
install the software, log out and back in.

## describe default namespace
`kubectl des<Tab> n<Tab><Tab> lfs458-<Tab>

kubectl -n kube-s<Tab> g<Tab> po<Tab>`

19. View other values we could have included in the kubeadm-config.yaml file when creating the cluster.

## This command prints objects such as the default init configuration that is used for 'kubeadm init'.
```
sudo kubeadm config print init-defaults

`apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
- system:bootstrappers:kubeadm:default-node-token
token: abcdef.0123456789abcdef
ttl: 24h0m0s
usages:
- signing
- authentication
kind: InitConfiguration
```


