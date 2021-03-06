<!-- Secure: Container Firewalls -->



This is Part 4 of 4 of the <a href="/guides/">Weave Cloud guides series</a>.
In this guide, how to secure your app by defining Kubernetes Network Policy and having it enforced by Weave Net is demonstrated.

<div style="width:50%; padding: 10px float:left;font-weight: 700;">
<a href="/guides/cloud-guide-part-3-monitor-prometheus-monitoring/">&laquo; Go to previous part: Part 3 – Monitor: Prometheus Monitoring</a>
</div>
<div style="clear:both;"></div>

<img src="images/secure.png" style="width:100%; border:1em solid #32324b;" />

<p></p>
Securing segments of your app is simple with the application of Kubernetes-based policy that is enforced by Weave Net. Add namespaces to the policy yaml files to create software firewalls and visualize the result in Weave Cloud.

<h2 id="a-video-overview">A Video Overview</h2>

<center><div style="width:530px; padding: 10px; display:inline-block; margin-top:2em;">
<iframe src="https://player.vimeo.com/video/190563581" width="640" height="360" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>
</div></center>


<h3 id="a-video-overview-sign-up-for-a-weave-cloud-account">Sign up for a Weave Cloud account</h3>

Go to [Weave Cloud](https://cloud.weave.works/) and register for an account.
You'll use the Weave Cloud token later to send metrics to Cortex and view Weave Net with it.

<img src="images/weave-cloud-token-1.png" style="width:100%;" />


<h2 id="deploy-a-kubernetes-cluster-with-weave-net-and-the-sample-app">Deploy a Kubernetes Cluster with Weave Net and the Sample App</h2>

If you have already done this as part of one of the other tutorials, you can skip this step.
Otherwise, click "Details" below to see the instructions.

XXX-START-DETAILS-BLOCK

**Note:** This example uses Digital Ocean, but you can just as easily create these three instances in AWS or whatever your favorite cloud provider is.

<h2 id="set-up-droplets-in-digital-ocean">Set Up Droplets in Digital Ocean</h2>

Sign up or log into [Digital Ocean](https://digitalocean.com) and create three Ubuntu 16.04 instances, where you'll deploy a Kubernetes cluster, add a container network using Weave Net and finally install the Sock Shop onto the cluster.

**Note:** It is recommended that each host have at least 4 gigabytes of memory in order to run this demo smoothly.

<h3 id="set-up-droplets-in-digital-ocean-create-three-ubuntu-instances">Create three Ubuntu Instances</h3>

Next, move over to Digital Ocean and create three Ubuntu 16.04 droplets. All of the machines should run Ubuntu 16.04 with 4GB or more of RAM per machine.

<h3 id="set-up-droplets-in-digital-ocean-adding-an-additional-instance-to-weave-cloud">Adding an Additional Instance to Weave Cloud</h3>

Sign up or log into [Weave Cloud](https://cloud.weave.works/).

Before you start installing Kubernetes, you may wish to [create an additional instance in Weave Cloud](https://cloud.weave.works/instances/create). This extra instance provides a separate "workspace" for this cluster, and in it you will be able to see the Sock Shop as it spins up on Kubernetes.

To create an additional instance, select the 'Create New Instance' command located in the menu bar.

<h2 id="set-up-a-kubernetes-cluster-with-kubeadm">Set up a Kubernetes Cluster with kubeadm</h2>

This is by far the simplest way in which to install Kubernetes.  With only a few commands, you will have deployed a complete Kubernetes cluster with a resilient and secure container network onto the Cloud Provider of your choice.

The installation uses a tool called `kubeadm` which is part of Kubernetes 1.4.

`kubeadm` works with local VMs, physical servers and/or cloud servers. It is simple enough that you can easily integrate it with your own automation (Terraform, Chef, Puppet, etc).

See the full [`kubeadm` reference](http://kubernetes.io/docs/admin/kubeadm) for information on all `kubeadm` command-line flags and for advice on automating `kubeadm` itself.

<h3 id="set-up-a-kubernetes-cluster-with-kubeadm-objectives">Objectives</h3>

* Install a secure Kubernetes cluster on your machines
* Install Weave Net as a pod network on the cluster so that application components (pods) can talk to each other
* Install a demo microservices application (a Socks Shop) onto the cluster
* View the result in Weave Cloud as you build the cluster

<h3 id="set-up-a-kubernetes-cluster-with-kubeadm-installing-kubelet-and-kubeadm-on-your-hosts">Installing kubelet and kubeadm on Your Hosts</h3>

Install the required packages on all the machines.

For each machine:

* SSH into the machine and become `root` if you are not already (for example, run `sudo su -`):

~~~
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
~~~

Install Docker if you don't have it already. You can also use the [official Docker packages](https://docs.docker.com/engine/installation/) instead of `docker.io` here.
~~~
apt-get install -y docker.io
~~~

Install the Kubernetes packages:
~~~
apt-get install -y kubelet kubeadm kubectl kubernetes-cni
~~~

<h3 id="set-up-a-kubernetes-cluster-with-kubeadm-initializing-the-master">Initializing the Master</h3>

**Note:** Before making one of your machines a master, `kubelet` and `kubeadm` must be installed onto each of the machines beforehand.

The master is the machine where the "control plane" components run, including `etcd` (the cluster database) and the API server (which the `kubectl` CLI communicates with).

All of these components run in pods started by `kubelet`.

Right now you can't run `kubeadm init` twice without turning down the cluster in between, see [Tear Down](#tear-down) for more information.

To initialize the master, pick one of the machines you previously installed `kubelet` and `kubeadm` on, and run:

~~~
kubeadm init
~~~

Initialization of the master may take a few minutes, so be patient.

**Note:** This will autodetect the network interface to advertise the master on as the interface with the default gateway.

If you want to use a different interface, specify `--api-advertise-addresses=<ip-address>` argument to `kubeadm init`.

Please refer to the [kubeadm reference doc](http://kubernetes.io/docs/admin/kubeadm/) if you want to read more about the flags `kubeadm init` provides.

The output should look like:

~~~
<master/tokens> generated token: "f0c861.753c505740ecde4c"
<master/pki> created keys and certificates in "/etc/kubernetes/pki"
<util/kubeconfig> created "/etc/kubernetes/kubelet.conf"
<util/kubeconfig> created "/etc/kubernetes/admin.conf"
<master/apiclient> created API client configuration
<master/apiclient> created API client, waiting for the control plane to become ready
<master/apiclient> all control plane components are healthy after 61.346626 seconds
<master/apiclient> waiting for at least one node to register and become ready
<master/apiclient> first node is ready after 4.506807 seconds
<master/discovery> created essential addon: kube-discovery
<master/addons> created essential addon: kube-proxy
<master/addons> created essential addon: kube-dns

Kubernetes master initialised successfully!
You can connect any number of nodes by running:
    kubeadm join --token <token> <master-ip>
~~~

Make a record of the `kubeadm join` command that `kubeadm init` outputs. You will need this in a moment.

The key included here is secret, keep it safe &mdash; anyone with this key can add authenticated nodes to your cluster.

The key is used for mutual authentication between the master and the joining nodes.

By default, the cluster does not schedule pods on the master for security reasons. If you want to be able to schedule pods on the master, for example if you want a single-machine Kubernetes cluster for development, run:

~~~
kubectl taint nodes --all dedicated-
~~~

The output will be:
~~~
node "test-01" tainted
taint key="dedicated" and effect="" not found.
taint key="dedicated" and effect="" not found.
~~~

This removes the "dedicated" taint from any nodes that have it, including the master node, meaning that the scheduler will then be able to schedule pods everywhere.

<h3 id="set-up-a-kubernetes-cluster-with-kubeadm-installing-weave-net">Installing Weave Net</h3>

In this section, you will install a Weave Net pod network so that your pods can communicate with each other.

**You must add Weave Net before deploying any applications to your cluster and before `kube-dns` starts up.**

Install [Weave Net](https://github.com/weaveworks/weave-kube) by logging in to the master and running:

~~~
kubectl apply -f https://git.io/weave-kube
~~~
The output will be:
~~~
daemonset "weave-net" created
~~~

**Note:** Install **only one** pod network per cluster.

Once a pod network is installed, confirm that it is working by checking that the `kube-dns` pod is `Running` in the output of `kubectl get pods --all-namespaces`.

And once the `kube-dns` pod is up and running, you can continue on to joining your nodes.

<h3 id="set-up-a-kubernetes-cluster-with-kubeadm-joining-your-nodes">Joining Your Nodes</h3>

The nodes are where your workloads (containers and pods, etc) run. If you want to add additional machines as nodes to your cluster, then SSH onto each new machine, become root (e.g. `sudo su -`) and run the command that was output by `kubeadm init`.

For example:

~~~
kubeadm join --token <token> <master-ip>
~~~

The output should look like:

~~~
<util/tokens> validating provided token
<node/discovery> created cluster info discovery client, requesting info from "http://X.X.X.X:9898/cluster-info/v1/?token-id=0f8588"
<node/discovery> cluster info object received, verifying signature using given token
<node/discovery> cluster info signature and contents are valid, will use API endpoints [https://X.X.X.X:443]
<node/csr> created API client to obtain unique certificate for this node, generating keys and certificate signing request
<node/csr> received signed certificate from the API server, generating kubelet configuration
<util/kubeconfig> created "/etc/kubernetes/kubelet.conf"
Node join complete:
* Certificate signing request sent to master and response
  received.
* Kubelet informed of new secure connection details.

Run `kubectl get nodes` on the master to see this machine join.
~~~

A few seconds later, you should notice that running `kubectl get nodes` on the master shows a cluster with as many machines as you created.

<h3 id="set-up-a-kubernetes-cluster-with-kubeadm-optional-control-your-cluster-from-machines-other-than-the-master">(Optional) Control Your Cluster From Machines Other Than The Master</h3>

In order to get kubectl on (for example) your laptop to talk to your cluster, copy the `kubeconfig` file from your master to your laptop:

~~~
scp root@<master ip>:/etc/kubernetes/admin.conf .
kubectl --kubeconfig ./admin.conf get nodes
~~~

<h3 id="set-up-a-kubernetes-cluster-with-kubeadm-install-and-launch-weave-scope">Install and Launch Weave Scope</h3>

Install and launch the Weave Scope probes onto your Kubernetes cluster. From the master:

~~~
curl -sSL 'https://cloud.weave.works/launch/k8s/weavescope.yaml?service-token=<YOUR_WEAVE_CLOUD_SERVICE_TOKEN>'|kubectl apply -f -
~~~

The `<YOUR_WEAVE_CLOUD_SERVICE_TOKEN>` is found in the settings dialog on [Weave Cloud](https://cloud.weave.works/).

Return to the Weave Cloud interface, select View Instance and click on the `Hosts`.

As you follow the next steps you can then watch the socks shop come up in [Weave Cloud](https://cloud.weave.works/).

Ensure that 'System Containers' are selected from the filters in the left hand corner to see all of the Kubernetes processes.

<img src="images/kubernetes-weave-cloud-3-1.png" style="width:100%;" />

<h3 id="set-up-a-kubernetes-cluster-with-kubeadm-installing-the-sock-shop-onto-kubernetes">Installing the Sock Shop onto Kubernetes</h3>

To put your cluster through its paces, install the sample microservices application, Socks Shop. To learn more about the sample microservices app, refer to the [microservices-demo README](https://github.com/microservices-demo/microservices-demo).

On the master, run:

~~~
kubectl create namespace sock-shop
git clone https://github.com/microservices-demo/microservices-demo
cd microservices-demo
kubectl apply -n sock-shop -f deploy/kubernetes/manifests
~~~

Switch to the `sock-shop` namespace at the bottom left of your browser window in Weave Cloud when in any of the Kubernetes-specific views (pods, replica sets, deployments & services).

<img src="images/sock-shop-kubernetes-1-1.png" style="width:100%;" />


<h3 id="set-up-a-kubernetes-cluster-with-kubeadm-viewing-the-sock-shop-in-your-browser">Viewing the Sock Shop in Your Browser</h3>

To find the port that the cluster allocated for the front-end service run:

~~~
kubectl describe svc front-end -n sock-shop
~~~

The output should look like:

~~~
Name:                   front-end
Namespace:              sock-shop
Labels:                 name=front-end
Selector:               name=front-end
Type:                   NodePort
IP:                     100.66.88.176
Port:                   <unset> 80/TCP
NodePort:               <unset> 31869/TCP
Endpoints:              <none>
Session Affinity:       None
~~~

It takes several minutes to download and start all of the containers, watch the output of `kubectl get pods -n sock-shop` to see when they're all up and running.

Or view the containers appearing on the screen as they get created in [Weave Cloud](https://cloud.weave.works/).

Launch the Sock Shop in your browser by going to the IP address of any of your node machines in your browser, and by specifying the NodePort. So for example, `http://<master_ip>:<pNodePort>`. You can find the IP address of the machines in the DigitalOcean dashboard.

In the example above, the NodePort was `31869`.

If there is a firewall, make sure it exposes this port to the internet before you try to access it.

<img src="images/socks-shop.png" style="width:100%;" />


<h3 id="set-up-a-kubernetes-cluster-with-kubeadm-run-the-load-test-on-the-cluster">Run the Load Test on the Cluster</h3>

After the Sock Shop has completely deployed onto the cluster, run a load test and view the results in Weave Cloud. You should see the architecture of the application emerge as different microservices begin communicating with one another.

~~~
docker run -ti --rm --name=LOAD_TEST  weaveworksdemos/load-test -r 10000 -c 20 -h <host-ip:[port number]>
~~~

Where,

* `<host-ip:[port number]>` is the IP of the master and the port number you see when you run `kubectl describe svc front-end -n sock-shop`


XXX-END-DETAILS-BLOCK


<h2 id="apply-a-network-policy-and-enforce-with-weave-net">Apply a Network Policy, and Enforce with Weave Net</h2>

In the above guide, you should have deployed the socks shop.  However, the different components are not isolated.

Let's start by testing that. Load up [Weave Cloud](https://cloud.weave.works/) and make sure you're in the containers view, then observe that the catalogue service can talk to the shipping service.

Search for the "catalogue" container and then click the `>_` icon from the details panel. This opens up a convenient shell for that container.  

<img src="images/catalogue-terminal.png" />

Type the following:

~~~
wget http://shipping
~~~

You should get:

~~~
wget: server returned error: HTTP/1.1 404
~~~

This is not good! The catalogue service can speak to the shipping service. For a hacker who managed to infiltrate the catalogue service, they could now get direct access to the shipping service and attack that too.

So, let's apply some network policy. SSH into the master, or where ever you run `kubectl`:

~~~
cd microservices-demo
kubectl apply -f deploy/kubernetes/manifests-policy/
~~~

Now run the wget inside the terminal in Weave Cloud again:

~~~
wget http://shipping
~~~

And you'll see the connection just times out. Those packets are being dropped. The app is now more secure!

You can [take a look at the network policy itself](https://github.com/microservices-demo/microservices-demo/tree/master/deploy/kubernetes/manifests-policy) and learn about [Kubernetes network policy](http://kubernetes.io/docs/user-guide/networkpolicies/) to learn how to write your own policy.

<h2 id="tear-down">Tear Down</h2>

XXX-START-DETAILS-BLOCK

Unless you are continuing onto another guide, or going to use the cluster for your own app, you may want to tear down the Sock Shop and also the Kubernetes cluster you created.

If you made a mistep during the install instructions, it is recommended that you delete the entire cluster and begin again.

* To uninstall the socks shop, run `kubectl delete namespace sock-shop` on the master.

* To uninstall Kubernetes on the machines, simply delete the machines you created for this tutorial, or run the script below and then start over or uninstall the packages.

* To uninstall a daemon set run `kubectl delete ds <agent-name>`. 

To reset local state run the following script:

~~~
systemctl stop kubelet;
docker rm -f -v $(docker ps -q);
find /var/lib/kubelet | xargs -n 1 findmnt -n -t tmpfs -o TARGET -T | uniq | xargs -r umount -v;
rm -r -f /etc/kubernetes /var/lib/kubelet /var/lib/etcd;
~~~

<h2 id="recreating-the-cluster-starting-over">Recreating the Cluster: Starting Over</h2>

If you wish to start over, run `systemctl start kubelet` followed by `kubeadm init` on the master and `kubeadm join` on any of the nodes.


XXX-END-DETAILS-BLOCK

<h2 id="conclusions">Conclusions</h2>

You've seen that Kubernetes network policy allows you to define flexible and dynamic security policies, and Weave Net allows you to enforce them.
<p></p>

If you have any questions or comments you can reach out to us on our <a href="https://weave-community.slack.com"> Slack channel </a> or through one of these other channels at <a href="https://www.weave.works/help/"> Help </a>.


<div style="width:50%; padding: 10px; float:left;font-weight: 700;">
<a href="/guides/cloud-guide-part-3-monitor-prometheus-monitoring/">&laquo; Go to previous part: Part 3 – Monitor: Prometheus Monitoring</a>
</div>
<div style="clear:both;"></div>

<p></p>
