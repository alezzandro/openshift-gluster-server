# Gluster server DaemonSet for OpenShift / Origin v3.2 
## Create HA self-service storage on OpenShift!


### Introduction
With this tutorial I'll show you how to setup an onboard Gluster service for OpenShift v3.

Thanks to the latest release of glusterd container by Red Hat: [here](https://www.redhat.com/it/technologies/storage/use-cases/container-native-storage) and [here](https://www.redhat.com/it/technologies/storage/use-cases/container-native-storage)

And the recent release of centos glusterd container version: [here](https://hub.docker.com/r/gluster/gluster-centos/)

<i>If you're interested in the RHEL7 based container you may use the file "<b>gluster-ds_rh.yml</b>" on your OpenShift Enterprise platform.</i><br>

I've just started playing with them and the great functionality: "DaemonSet", introduced in version OpenShift 3.2.
https://blog.openshift.com/openshift-3-2-whats-new/


In the following steps I'll show you how to setup a glusterd privileged container on one or more OpenShift managed node using local storage to serve replicated and highly available storage for you OpenShift deployments.

This privileged container will mount a mountpoint on your openshift node and access to host real network. For that matter you should ensure that the container does not conflict with any already used port it may needs. I've also setup an ansible playbook that opens up the required ports on the chosen (default all) openshift nodes.

At end you'll have one or multiple replicated gluster volume that you can mount in your running pod and/or you can use for creating Persistent Volumes (pv) accessed by pod through Persistent Volume Claim (pvc).

<b>PLEASE NOTE:</b>

<b><i>The following steps are not supported in any way by Red Hat and they are not production-ready.</i></b>
<br>
<br>
### Configuration

First of all take clone my git repository to download all the required file:
```
# git clone https://github.com/alezzandro/openshift-gluster-server
```

Run the init.yml playbook in case you need to setup your managed nodes (directories creation and firewall rules setup):
```
# ansible-playbook init.yml
```

Create a new project for holding the new gluster pods that we'll create:
```
# oc new-project gluster
```

Select the project for creating new resources:
```
# oc project gluster
```

Create a new service account for handling privileged container deployment on the gluster project and assign the right permission to that sa:
```
# oc create -f gluster-sa.yml
# oadm policy add-scc-to-user privileged system:serviceaccount:gluster:gluster
```

Now we need to properly label the nodes that will run the glusterd containers (please change the commands accordingly):
```
# oc label node pocosegen3.gen.local app=gluster
# oc label node pocosegen4.gen.local app=gluster
# oc label node pocosegen5.gen.local app=gluster
```

Noew it's time to create the Gluster DaemonSet entity:
```
# oc create -f gluster-ds.yml
```

Then we can monitor the creation with command:
```
# oc get pods -w
```

Just for example I'll show you the result on my environment:
```
[root@pocosegen1-v2 gluster]# oc get pods
NAME            READY     STATUS    RESTARTS   AGE
gluster-bthb0   1/1       Running   0          17m
gluster-oo3e8   1/1       Running   0          2h
gluster-wsek1   1/1       Running   0          2h
```

After that we can then access to one of the created container to start the peer probing process:

<i>[Please note the ip addresses I'll use are the real ip addresses of two of my managed nodes]</i>
```
# oc rsh gluster-bthb0
# gluster peer probe 172.18.212.29
# gluster peer probe 172.18.212.10
```

Then we can check the peer status (on the container itself):
```
# gluster peer status
```

After that we can create one test gluster volume and start it (on the container itself):
```
# gluster volume create testvolume replica 3 172.18.212.28:/mnt/brick1/testvolume 172.18.212.29:/mnt/brick1/testvolume 172.18.212.10:/mnt/brick1/testvolume 
# gluster volume start testvolume
```

That's all!

You can now use the just created gluster volume on your OpenShift projects or also outside.
<br><br>
### OpenShift Gluster Volume Usage

You can use the just setup Gluster volume for your OpenShift deployments. 

For that purpose I've uploaded under the template directory two files "gluster-endpoints.yaml" and "gluster-service.yaml".

These two file can be used to setup the gluster service used for balancing the gluster endpoints you just configured.

Please don't forget to customize the endpoints file before creating the resources on your chosen project.

For more information and example please take a look at: https://github.com/openshift/origin/tree/master/examples/storage-examples/gluster-examples
