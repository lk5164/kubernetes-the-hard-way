# Provisioning Compute Resources

## Set's IP addresses in the range 192.168.1.0/24
<table>
 <tr>
 <th>Name</th><th>	IP address</th><th>Role</th>
 </tr>
 <tr>
  <td>kube-haproxy</td><td>192.168.1.24</td><td>Load Balancer</td>
 </tr>
 <tr>
  <td>kube-controller0</td><td>192.168.1.25</td><td>controller node</td>
 </tr>
 <tr>
  <td>kube-controller1</td><td>192.168.1.17</td><td>controller node</td>
 </tr>
 <tr>
  <td>kube-worker0</td><td>192.168.1.23</td><td>worker node</td>
 </tr>
 <tr>
  <td>kube-worker1</td><td>192.168.1.18</td><td>worker node</td>
 </tr>
</table>

If you don't have DNS, edit `/etc/hosts` file on each VM machines with the hosts' information. 

- Install's Docker on Worker nodes
- Runs the below command on all nodes to allow for network forwarding in IP Tables.
  This is required for kubernetes networking to function correctly.
    > sysctl net.bridge.bridge-nf-call-iptables=1


## SSH to the nodes

    $: ssh-keygen -t rsa -f ~/.ssh/id_rsa -P '' && \
    cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

Copy config file to .ssh folder. 
    
    cp ssh_config ~/.ssh/

Copying keys to slave machines is not required step. However when you execute start-dfs.sh or start-yarn.sh command, you need to manually input password. There are various way of copying public keys to other machine. I am using ssh-copy-id command, 
  
    ssh-copy-id [username]@[ip]

## Verify Environment

- Ensure all VMs are up
- Ensure VMs are assigned the above IP addresses
- Ensure you can SSH into these VMs using the IP and private keys
- Ensure the VMs can ping each other
- Ensure the worker nodes have Docker installed on them. Version: 18.06
  > command `sudo docker version`

## Troubleshooting Tips

If you have vCenter installed, you can create vms via templates. However, you may get issue if try to create a vm with docker installed. I would recommend creating a vm template before Docker installed. 

If any of the VMs failed to provision, or is not configured correct, delete the vm and recreate from template

Next: [Installing the Client Tools](03-client-tools.md)

