---
title: "Kubernetes Tips - Basic Network Debugging"
layout: post
date: 2020-11-30
categories: 
  - "aws"
  - "azure"
  - "containers"
  - "devops"
  - "gcp"
tags: 
  - "aks"
  - "aws"
  - "azure"
  - "cloud"
  - "containers"
  - "devops"
  - "eks"
  - "gcp"
  - "gke"
  - "kubernetes"
  - "microservices"
  - "networking"
---

If, like me, you've come from a traditional sysadmin background then Kubernetes can be daunting to say the least, this doesn't get much easier when it comes to trying to get to grips with how to debug networking issues. Kubernetes networking is **VAST** and supports a number of complex implementations that vary between the major Kubernetes-as-a-Service platforms (GKE, EKS, AKS) as well as many other options. The broad strokes are broken down in the **[official docs](https://kubernetes.io/docs/concepts/cluster-administration/networking/)** but as is so often the case the manual won't do much to help you when it comes to crunch time and you have a networking issue to solve.

The rules for basic network debugging haven't changed that much however and in this post I want to take a look at some simple methods for using some of the old standards for networking troubleshooting and how we can leverage them within a Kubernetes Cluster to make life easier.

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/k8s.png" class="scaled-img-50">
  <figcaption>So many packets but how to read them?</figcaption>
</figure>

## Busybox and **Netshoot** - Weapons of Choice

Most admins are familiar at least with [**busybox**](https://hub.docker.com/_/busybox), the Swiss Army Knife of Linux troubleshooting; it comes with dozens of binaries for debugging, several of which are handy for network issues, a much newer and more network-focused project however is **[netshoot](https://github.com/nicolaka/netshoot)** which contains a **HUGE** amount of tools for debugging applications at just about every points of the stack. I won't be looking at **netshoot** for the basics but we'll come back to it for some more advanced stuff.

For our purposes we'll be looking at some vital tools inside **busybox**:

- telnet
- tcpdump
- dig/nslookup
- traceroute

The first thing we'll need to do before anything is get a working instance of **busybox** inside out Kubernetes Cluster, we can do this using **kubectl** **run** as below:

```bash
# Deploy to the default namespace
kubectl run -i --tty mydebugpod --image=busybox --restart=Never -- sh
# If you don't see a command prompt, try pressing enter.
/#

# Deploy to a specific namespace
kubectl run -i --tty mydebugpod -n NAMESPACE --image=busybox --restart=Never -- sh
# If you don't see a command prompt, try pressing enter.
/#

```

This will deploy a _pod_ named **mydebugpod** based on the official **busybox** image (which we will pull directly from DockerHub unless otherwise configured). The **\-- sh** argument will pass us directly in to the shell of the pod once it has started.

From here we can use the tools included within **busybox**.

### Debugging Egress with Telnet

Now that we have **busybox** running within the same namespace as our pod, we can test egress to another node using **telnet** exactly as we would in a traditional environment:

```bash
telnet testserver.tinfoilcipher.co.uk 22
# connected to testserver.tinfoilcipher.co.uk.
# Escape character is '^]'.
# SSH-2.0-OpenSSH_7.2p2 Ubuntu-4ubuntu2.8

```

From this we can see that we have egress from our _namespace_ to our node over SSH, this same method can be used to debug TCP connectivity to any IP address or hostname and is a simple test to perform to test egress from a _namespace_.

### Testing DNS Resolution

DNS resolution is another common issue which we can test with **busybox** using **dig** or **nslookup**:

```bash
# Forward lookup
dig testserver.tinfoilcipher.co.uk
nslookup testserver.tinfoilcipher.co.uk

# Reverse lookup
dig 10.0.1.10
nslookup 10.0.1.10
```

### Route Testing

In order to confirm that traffic is observing the expected route through your Cluster (in the event you have a complex configuration) and after existing the cluster, **busybox** contain traceroute, we can use this to test that our traffic is going the right way:

```bash
traceroute testserver.tinfoilcipher.co.uk
```

## Packet Captures - The Ultimate Tool

This is all good and well for very basic tests, however sooner or later the only thing that's going to do for a complicated problem is a proper packet capture. As most admins know there's three tools you really have at your disposal; **[Wireshark](https://www.wireshark.org/)**, **[tshark](https://www.wireshark.org/docs/man-pages/tshark.html)** (the terminal driven deployment of Wireshark) and **[tcpdump](https://www.tcpdump.org/)**.

**Wireshark** is the undisputed king of network troubleshooting tools, but getting it to play with Kubernetes is something of a challenge, luckily the packet captures from **tcpdump** can be exported and viewed in Wireshark, so our problem gets a little easier).

## Packet Captures - Available...With Caveats

A great new feature was introduced in Kubernetes at 1.16 as an alpha called _[**Ephemeral Debug Containers**](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-running-pod/#ephemeral-container)_ which allows for the injection of a debugging sidecar directly in to a running _pod_, but since I spend most of my time working in EKS and at the time of writing this functionality hasn't yet to be added (and doesn't look set to for some time..so I'll have to find another option). We'll have to come back to _Debug Containers_ in another post.  
  
Our problem can still be solved but right out of the gate but **I wouldn't recommend this for anything but a development environment** **and certainly wouldn't recommend it for production!**

To get our capture, we can attempt to install **tcpdump** DIRECTLY in to a running container. This is a very variable process and relies on our container being built on a base image that has both an interactive shell and some package management (Alpine, Ubuntu etc.). Hardened "distroless" images aren't a candidate for this process.

So you want to try this anyway? Let's get an interactive shell on a running pod, in my example the _container_ is based on Alpine Linux and uses the **apk** package manager:

```bash
kubectl exec --stdin --tty tinfoilapp-497d0a86e0-nvhea -c tinfoil -n tinfoiltest -- /bin/sh
# If you don't see a command prompt, try pressing enter.

/# apk add tcpdump
/# exit
```

This will install **tcpdump** in to our running container (**tinfoil**). This is obviously far from perfect as we have modified a live container which we really shouldn't be doing, but it will get us where we need to be for the purposes of emergency troubleshooting, from here our command to capture traffic changes slightly as we will need to use **kubectl exec** to execute **tcpdump** within our container and pass the traffic to either **Wireshark** or **tshark** (depending on our configuration):

```bash
#--Capture in Wireshark (Linux)
kubectl exec tinfoilapp-497d0a86e0-nvhea -c tinfoil -n tinfoiltest -- tcpdump -i eth0 -w - | wireshark -k -i -

#--Capture in tshark and export pcap (WSL on Windows)
kubectl exec tinfoilapp-497d0a86e0-nvhea -c tinfoil -n tinfoiltest -- tcpdump -i eth0 -w - | sudo tshark -i - -w /mnt/c/TEMP/testcapture.pcap
```

The Linux version will stream the **tcpdump** data directly in to Wireshark, it's pretty cool to see, the Windows version is a lot more restrictive, that last pipe just doesn't seem to place properly with Windows when using WSL. But we can export the pcap to the file system and then open in it in Wireshark in Windows just fine.

Finally, make sure you clean up your running pod:

```bash
kubectl exec --stdin --tty tinfoilapp-497d0a86e0-nvhea -c tinfoil -n tinfoiltest -- /bin/sh
# If you don't see a command prompt, try pressing enter.

/# apk del tcpdump
/# exit
```

It's probably more prudent at this point, however, to simply re-provision the entire _pod_ for safety, especially if you've had to do this in a production environment!

## Conclusion

After a couple of weeks of digging in to the topic, I've found that getting to grips with the basics isn't as bad as it looks. Hopefully this helps a few people be less intimidated by debugging Kubernetes networking issues, the basics never really change all that much when trying to solve simple networking issues.

Whilst Kubernetes networking is **FAR** from simple (and some deployments really do make my head spin), the small issues can often still be small and having the right tools to hand to try and observe and solve them them makes life much easier than trying to work out problems with a blindfold on :).
