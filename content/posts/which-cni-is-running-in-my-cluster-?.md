+++
title = 'Which Cni Is Running in My Cluster ?'
date = 2024-07-23T22:28:12+02:00
draft = false
tags = ['kubernetes', 'networking']
+++

Today a colleague of mine who is new to Kubernetes asked me what was the CNI running on our production cluster, I knew the answer but I wanted to doublecheck, just to see if I could climb up to the source of truth. 

My first reflex was to search in the kube-system namespace for pods with a CNI-like name. And I quickly found what I was looking for :

![cni-1](/which-cni-1.png)

With this I was pretty sure **Antrea** was the chosen CNI, but I knew there were other ways to verify it. 

When a CNI is installed on a cluster you can usually find its binaries somewhere on the node, they are generally located under the `/opt/cni` directory :

![cni-4](/which-cni-4.png)

The binaries are there, but we can see that we have `antrea` as well as `flannel` binaries which are two completely different CNIs, we need to check which of them are used by the kubelet when creating the CNI pods in the cluster.

We can do this by checking the `/etc/cni/net.d` directory that normaly hosts the config file for the cluster's chosen CNI :

![cni-2](/which-cni-2.png)
The problem here is that we have two configuration files, one for flannel and one for antrea which is quite logical taking in account that we previously checked the CNI binaries directory and found both of these CNIs there. Having two configuration files here doesn't help us because we can not determine which one is used and thus which is the CNI used by the cluster. 

The thing we have to remember here is that the ***kubelet always execute the first configuration file it finds under the `/etc/cni/net.d` directory***.

We thus simply have to check what is the first configuration file that will be found by the kubelet while inspecting the `net.d` directory :

![cni-3](/which-cni-3.png)

As expected its the `antrea` config file that will be executed by the kubelet, we are now 100% sure that Antrea is the CNI running in our cluster.

It is also possible to check and analyse the logs of the Kubelet in order to find logs related to the CNI with `journalctl -u kubelet | less`, then search for the pattern or `journalctl -u kubelete | grep <pattern>`
