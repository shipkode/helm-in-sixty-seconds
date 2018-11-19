# Helm-in-sixty-seconds

## Getting started

This automation runs on Mac OSX. Before you get started, you will need to
install the pre-requisites described below.

### Installing pre-requisites

* [Virtualbox](https://www.virtualbox.org/) ([download](https://www.virtualbox.org/wiki/Downloads)|[install](https://www.virtualbox.org/manual/ch02.html#idm861))
* [Vagrant](https://www.vagrantup.com) ([download](https://www.vagrantup.com/downloads.html)|[install](https://www.vagrantup.com/docs/installation/))
* [Ansible](https://www.ansible.com/) ([install](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html))

### Checking out the code

To clone this repo, use the following command:

    git clone git@github.com:shipkode/helm-in-sixty-seconds.git

At this point, you are ready to get started!

## Running

### Creating your VM

You now are ready to standup your VM. Type in the commands below:

    cd helm-in-sixty-seconds
    vagrant up

That's it! Now watch as the wonders of automation unfold, and your
environment and workloads are all setup.

Once the VM has been provisioned and the automation has completed, you
should see something like this:

    TC", "WatchdogTimestampMonotonic": "396883265", "WatchdogUSec": "0"}}
    META: ran handlers
    META: ran handlers

    PLAY RECAP *********************************************************************
    helm1                      : ok=60   changed=33   unreachable=0    failed=0

Check the current VM status:

    MacBook-Pro:ansible dlapsley$ vagrant status
    Current machine states:

    helm1                     running (virtualbox)

    The VM is running. To stop this VM, you can run `vagrant halt` to
    shut it down forcefully, or you can run `vagrant suspend` to simply
    suspend the virtual machine. In either case, to restart it again,
    simply run `vagrant up`.

### Check Cluster Status

Let's ssh into the VM and check it out:

    MacBook-Pro:ansible dlapsley$ vagrant ssh

    Welcome to Ubuntu 18.04.1 LTS (GNU/Linux 4.15.0-36-generic x86_64)

     * Documentation:  https://help.ubuntu.com
     * Management:     https://landscape.canonical.com
     * Support:        https://ubuntu.com/advantage

      System information as of Mon Nov 19 00:38:33 UTC 2018

      System load:  0.46              Users logged in:        0
      Usage of /:   3.3% of 96.88GB   IP address for enp0s3:  10.0.2.15
      Memory usage: 10%               IP address for docker0: 172.17.0.1
      Swap usage:   0%                IP address for cni0:    10.244.0.1
      Processes:    137


      Get cloud support with Ubuntu Advantage Cloud Guest:
        http://www.ubuntu.com/business/services/cloud

    54 packages can be updated.
    25 updates are security updates.


    Last login: Sun Nov 18 23:46:29 2018 from 10.0.2.2
    vagrant@helm1:~$

Let's have a look at the current status of Persistent Volumes (pv),
Persistent Volume Claims (pvc), pods, and storage classes:

    vagrant@helm1:~$ watch kubectl get pv,pvc,pod,storageclass --all-namespaces

    Every 2.0s: kubectl get pv,pvc,pod,storageclass --all-namespaces                                                       helm1: Mon Nov 19 00:39:41 2018
    
    NAME                                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM     STORAGECLASS    REASON    AGE
    persistentvolume/local-storage-pv   10Gi       RWO            Retain           Available             local-storage             53m

    NAMESPACE   NAME                                        STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS    AGE
    default     persistentvolumeclaim/local-storage-claim   Pending                                       local-storage   53m

    NAMESPACE     NAME                                 READY     STATUS    RESTARTS   AGE
    kube-system   pod/coredns-78fcdf6894-69d9w         1/1       Running   0          53m
    kube-system   pod/coredns-78fcdf6894-kz7wb         1/1       Running   0          53m
    kube-system   pod/etcd-helm1                       1/1       Running   0          52m
    kube-system   pod/kube-apiserver-helm1             1/1       Running   0          52m
    kube-system   pod/kube-controller-manager-helm1    1/1       Running   0          52m
    kube-system   pod/kube-flannel-ds-amd64-jhl5r      1/1       Running   0          53m
    kube-system   pod/kube-proxy-tsnvc                 1/1       Running   0          53m
    kube-system   pod/kube-scheduler-helm1             1/1       Running   0          52m
    kube-system   pod/tiller-deploy-7679459f98-sxnxk   1/1       Running   0          53m

    NAMESPACE   NAME                                        PROVISIONER                    AGE
                storageclass.storage.k8s.io/local-storage   kubernetes.io/no-provisioner   53m

### Check Helm Installation

As we can see, tiller is installed. Let's see what version of helm we are running:

    vagrant@helm1:~$ helm version
    Client: &version.Version{SemVer:"v2.11.0", GitCommit:"2e55dbe1fdb5fdb96b75ff144a339489417b146b", GitTreeState:"clean"}
    Server: &version.Version{SemVer:"v2.11.0", GitCommit:"2e55dbe1fdb5fdb96b75ff144a339489417b146b", GitTreeState:"clean"}

### Install sample application using Helm

Now, we'll install a sample application using helm (in this case mysql):

    vagrant@helm1:~$  helm install stable/mysql --set persistence.existingClaim=local-storage-claim
    NAME:   dapper-warthog
    LAST DEPLOYED: Mon Nov 19 00:40:19 2018
    NAMESPACE: default
    STATUS: DEPLOYED

    RESOURCES:
    ==> v1/Secret
    NAME                  AGE
    dapper-warthog-mysql  0s
    
    ==> v1/ConfigMap
    dapper-warthog-mysql-test  0s

    ==> v1/Service
    dapper-warthog-mysql  0s

    ==> v1beta1/Deployment
    dapper-warthog-mysql  0s

    ==> v1/Pod(related)

    NAME                                   READY  STATUS    RESTARTS  AGE
    dapper-warthog-mysql-659d77d749-ghjd7  0/1    Init:0/1  0         0s


    NOTES:
    MySQL can be accessed via port 3306 on the following DNS name from within your cluster:
    dapper-warthog-mysql.default.svc.cluster.local

    To get your root password run:

        MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default dapper-warthog-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)

    To connect to your database:

    1. Run an Ubuntu pod that you can use as a client:

        kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il

    2. Install the mysql client:

        $ apt-get update && apt-get install mysql-client -y

    3. Connect using the mysql cli, then provide your password:
        $ mysql -h dapper-warthog-mysql -p

    To connect to your database directly from outside the K8s cluster:
        MYSQL_HOST=127.0.0.1
        MYSQL_PORT=3306

        # Execute the following command to route the connection:
        kubectl port-forward svc/dapper-warthog-mysql 3306

        mysql -h ${MYSQL_HOST} -P${MYSQL_PORT} -u root -p${MYSQL_ROOT_PASSWORD}

Let's take a look at the state of the cluster:

    vagrant@helm1:~$ watch kubectl get pv,pvc,pod,storageclass --all-namespaces

    NAME                                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                         STORAGECLASS    REASON    AGE
    persistentvolume/local-storage-pv   10Gi       RWO            Retain           Bound     default/local-storage-claim   local-storage             55m

    NAMESPACE   NAME                                        STATUS    VOLUME             CAPACITY   ACCESS MODES   STORAGECLASS    AGE
    default     persistentvolumeclaim/local-storage-claim   Bound     local-storage-pv   10Gi       RWO            local-storage   55m

    NAMESPACE     NAME                                        READY     STATUS    RESTARTS   AGE
    default       pod/dapper-warthog-mysql-659d77d749-ghjd7   1/1       Running   0          1m
    kube-system   pod/coredns-78fcdf6894-69d9w                1/1       Running   0          56m
    kube-system   pod/coredns-78fcdf6894-kz7wb                1/1       Running   0          56m
    kube-system   pod/etcd-helm1                              1/1       Running   0          55m
    kube-system   pod/kube-apiserver-helm1                    1/1       Running   0          55m
    kube-system   pod/kube-controller-manager-helm1           1/1       Running   0          55m
    kube-system   pod/kube-flannel-ds-amd64-jhl5r             1/1       Running   0          56m
    kube-system   pod/kube-proxy-tsnvc                        1/1       Running   0          56m
    kube-system   pod/kube-scheduler-helm1                    1/1       Running   0          55m
    kube-system   pod/tiller-deploy-7679459f98-sxnxk          1/1       Running   0          55m

    NAMESPACE   NAME                                        PROVISIONER                    AGE
                storageclass.storage.k8s.io/local-storage   kubernetes.io/no-provisioner   55m

## License

Apache Software License 2.0.

## References

* [Mysql Helm Chart](https://github.com/helm/charts/tree/master/stable/mysql)
* [Kubernetes Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/#local)
* [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#claims-as-volumes)
* [Kubernetes Local Volumes](https://kubernetes.io/blog/2018/04/13/local-persistent-volumes-beta/)
* [Kubernetes Claims as Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#claims-as-volumes)
* [Kubernetes Configure Persistent Volume Storage](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage)
