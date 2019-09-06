# Provisioning Compute Resources

## Set's IP addresses in the range 192.168.1.0/24
<table>
 <tr>
 <th>Name</th><th>	IP address</th><th>Role</th>
 </tr>
 <tr>
  <td>haproxy</td><td>192.168.1.24</td><td>Load Balancer</td>
 </tr>
 <tr>
  <td>controller0</td><td>192.168.1.25</td><td>controller node</td>
 </tr>
 <tr>
  <td>controller1</td><td>192.168.1.17</td><td>controller node</td>
 </tr>
 <tr>
  <td>worker0</td><td>192.168.1.23</td><td>worker node</td>
 </tr>
 <tr>
  <td>worker1</td><td>192.168.1.18</td><td>worker node</td>
 </tr>
</table>

- Install's Docker on Worker nodes
- Runs the below command on all nodes to allow for network forwarding in IP Tables.
  This is required for kubernetes networking to function correctly.
    > sysctl net.bridge.bridge-nf-call-iptables=1


## SSH to the nodes


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

