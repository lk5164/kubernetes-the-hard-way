# Prerequisites
## Infrastructure Design
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
	
Operating System: Ubuntu 18.04 LTS (VMs on top of VMware ESXi)
Networking solution: Weave Net
DNS Add-on - coredns
