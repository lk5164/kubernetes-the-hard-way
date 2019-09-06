# Prerequisites
Esxi: 6.7 U2
	
Operating System: Ubuntu 18.04 LTS (VMs on top of VMware ESXi)

Networking solution: Weave Net

DNS Add-on - coredns

## Running Commands in Parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. Labs in this tutorial may require running the same commands across multiple compute instances, in those cases consider using tmux and splitting a window into multiple panes with `synchronize-panes` enabled to speed up the provisioning process.

> The use of tmux is optional and not required to complete this tutorial.

![tmux screenshot](https://raw.githubusercontent.com/mmumshad/kubernetes-the-hard-way/master/docs/images/tmux-screenshot.png)

> Enable `synchronize-panes`: `ctrl+b` then `shift :`. Then type `set synchronize-panes on` at the prompt. To disable synchronization: `set synchronize-panes off`.

Next: [Installing the Client Tools](02-client-tools.md)
