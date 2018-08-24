<!--
  Copyright (c) Microsoft Corporation
  All rights reserved.

  MIT License

  Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated
  documentation files (the "Software"), to deal in the Software without restriction, including without limitation
  the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and
  to permit persons to whom the Software is furnished to do so, subject to the following conditions:
  The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

  THE SOFTWARE IS PROVIDED *AS IS*, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING
  BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
  NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM,
  DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
  OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
-->

# Framework Launcher On Kubernetes User Manual

This guide introduces how to submit deep learning frameworks by framework launcher on K8S.

## Table of Contents:

- [Simple Tensorflow Framework Example](#simpletf)
- [Tensorflow Framework Example with Pod Initialization and Supporting Volumes](#tfinitvolume)
  - [Create a Pod that has an Init Container](#initcontainer)
    - [Configure Docker Image for Init Container](#dockerimginitcontainer)
    - [Configure a Pod with Init Container](#podinitcontainer)
  - [Configure a Pod to Use NFS Volume](#nfsvolume)
    - [Set up a NFS Server](#nfsserver)
    - [Mount on NFS Volume](#nfsmount)
  - [Configure a Pod to Use hostPath Volume](#hostpathvolume)
    - [Single Node using hostPath](#singlenodehp)
    - [Multiple Nodes using hostPath](#multinodehp)
  - [Tensorflow Framework Examples](#tfexamples)
    - [Tensorflow Framework using NFS](#tfnfs)
    - [Tensorflow Framework using hostPath](#tfhostpath)


## Simple Tensorflow Framework Example <a name="simpletf"></a>

To submit a Tensorflow Framework, you need to prepare a job configuration file and post it to APIServer.

`POST to http://{ApiServerAddress}/apis/launcher.microsoft.com/v1/namespaces/default/frameworks`

Here is one sample Tensorflow Framework configuration file:

```yaml
apiVersion: launcher.microsoft.com/v1
kind: Framework
metadata:
  name: tfexample
spec:
  description: "tf example"
  executionType: Start
  retryPolicy:
    fancyRetryPolicy: false
    maxRetryCount: 3
  taskRoles:
    - name: worker
      taskNumber: 1
      frameworkAttemptCompletionPolicy:
        minFailedTaskCount: 1
        minSucceededTaskCount: -1
      task:
        retryPolicy:
          fancyRetryPolicy: false
          maxRetryCount: 3
        pod:
          spec:
            restartPolicy: Never
            containers:
              - name: tf
                image: "zichengfang/k8s_launcher:distributedtf"
                ports:
                - containerPort: 49866
                workingDir: /benchmarks/scripts/tf_cnn_benchmarks
                command: ["/bin/bash","-c","export framework_name=tf-example && source ./get_ips.sh && python tf_cnn_benchmarks.py --batch_size=32 --model=alexnet --local_parameter_device=cpu --variable_update=parameter_server --ps_hosts=$ps_hosts:49867 --worker_hosts=$worker_hosts:49866 --job_name=worker --task_index=0 --data_name=cifar10 --data_dir=./data/cifar-10-batches-py/ --data_format=NHWC"]
            hostNetwork: true
    - name: ps
      taskNumber: 1
      frameworkAttemptCompletionPolicy:
        minFailedTaskCount: 1
        minSucceededTaskCount: -1
      task:
        retryPolicy:
          fancyRetryPolicy: false
          maxRetryCount: 3
        pod:
          spec:
            restartPolicy: Never
            containers:
              - name: tf
                image: "zichengfang/k8s_launcher:distributedtf"
                ports:
                - containerPort: 49867
                workingDir: /benchmarks/scripts/tf_cnn_benchmarks
                command: ["/bin/bash","-c","export framework_name=tf-example && source ./get_ips.sh && python tf_cnn_benchmarks.py --batch_size=32 --model=alexnet --local_parameter_device=cpu --variable_update=parameter_server --ps_hosts=$ps_hosts:49867 --worker_hosts=$worker_hosts:49866 --job_name=ps --task_index=0 --data_name=cifar10"]
            hostNetwork: true
```

*NOTE: you need to set hostNetwork: true if you use hotsNetwork.*

## Tensorflow Framework Example with Pod Initialization and Supporting Volumes <a name="tfinitvolume"></a>

### Create a Pod that has an Init Container <a name="initcontainer"></a>

In this example you create a Pod that has one application Container and one Init Container. The init container runs to completion before the application container starts. The init container writes worker and ps hostIPs into a configure file, and the application container can read this file (we will introduce how to use volume to write and read config file in later part).

#### Configure Docker Image for Init Container <a name="dockerimginitcontainer"></a>

You need to build an image for Init Container.

Here is the Dockerfile:

```dockerfile
FROM ubuntu:16.04

# Install CURL
RUN apt-get update
RUN apt-get install -y curl

# Install kubectl v1.10.0
RUN curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.10.0/bin/linux/amd64/kubectl && chmod +x ./kubectl && mv ./kubectl /usr/local/bin/kubectl

# Copy kubectl config file
COPY kubectlconfig .

# Copy get_ips.sh
COPY get_ips.sh .
```
Here is exmaple of kubectlconfid:

```
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: {YourKubernetesAPIServerAddress}
  name: test
contexts:
- context:
    cluster: example
    user: example
  name: example
current-context: example
kind: Config
preferences: {}
users:
- name: exampleuser
  user:
    password: {Password}
    username: {Username}
```

You can check [get_ips.sh](./example/buildInitContainerImage/get_ips.sh).

Then, to compile your image just run:

```bash
$ docker build -t yourInitConatinerImageName:version .
```

and you need to push this image to Dockerhub:

```bash
$ docker push yourInitConatinerImageName:version
```

#### Configure a Pod with Init Container <a name="podinitcontainer"></a>

Now you have already built your Init Conatiner Image. You need to configure pod to use Init Container.

Here is the configuration file for the Pod:

```yaml
apiVersion: launcher.microsoft.com/v1
kind: Framework
metadata:
  name: tfexample
spec:
  description: "tf example"
  executionType: Start
  retryPolicy:
    fancyRetryPolicy: false
    maxRetryCount: 3
  taskRoles:
    - name: worker
      taskNumber: 1
      frameworkAttemptCompletionPolicy:
        minFailedTaskCount: 1
        minSucceededTaskCount: -1
      task:
        retryPolicy:
          fancyRetryPolicy: false
          maxRetryCount: 3
        pod:
          spec:
            restartPolicy: Never
            containers:
              - name: tf
                # image: yourtensorflowjobimage:version
                image: zichengfang/k8s_launcher:distributedtf     
                ports:
                - containerPort: 49866
                workingDir: /benchmarks/scripts/tf_cnn_benchmarks
                command: ["/bin/bash","-c","cat /configdir/configfile"]
                olumeMounts:
                  - name: config-volume
                    mountPath: /configdir
            initContainers:
              - name: initips
                # image: yourInitConatinerImageName:version
                image: zichengfang/k8s_initcontainer:update            
                ports:
                - containerPort: 49866
                workingDir: /
                command: ["/bin/bash","-c","export worker_tasknum=1 && export ps_tasknum=1 && export ps_port=46867 && export worker_port=46866 && source ./get_ips.sh && echo -e $worker_hosts'\n'$ps_hosts > /configdir/configfile"]
                volumeMounts:
                  - name: config-volume
                    mountPath: /configdir
            volumes:
              - name: config-volume
                emptyDir: {}
            hostNetwork: true
    - name: ps
      taskNumber: 1
      frameworkAttemptCompletionPolicy:
        minFailedTaskCount: 1
        minSucceededTaskCount: -1
      task:
        retryPolicy:
          fancyRetryPolicy: false
          maxRetryCount: 3
        pod:
          spec:
            restartPolicy: Never
            containers:
              - name: tf
                # image: yourtensorflowjobimage:version
                image: zichengfang/k8s_launcher:distributedtf
                ports:
                - containerPort: 49867
                workingDir: /benchmarks/scripts/tf_cnn_benchmarks
                command: ["/bin/bash","-c","cat /configdir/configfile"]
                volumeMounts:
                  - name: config-volume
                    mountPath: /configdir
            initContainers:
              - name: initips
                # image: yourInitConatinerImageName:version
                image: zichengfang/k8s_initcontainer:update
                ports:
                - containerPort: 49867
                workingDir: /
                command: ["/bin/bash","-c","export worker_tasknum=1 && export ps_tasknum=1 && export ps_port=46867 && export worker_port=46866 && source ./get_ips.sh && echo -e $worker_hosts'\n'$ps_hosts > /configdir/configfile"]
                volumeMounts:
                  - name: config-volume
                    mountPath: /configdir
            volumes:
              - name: config-volume
                emptyDir: {}
            hostNetwork: true
```

Compared to simple Tensorflow Framework configuration file, you add `taskRole.task.pod.spec.initContainers`, `taskRole.task.pod.spec.containers.volumeMounts`, and `taskRole.task.pod.spec.volumes`. 

Below please find the detailed explanation for each of the parameters of `initConatiners`:

| Filed Name      | Description                                                                                                                                         |
|-----------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| `name`          | Name of the initContainer, string in `^[a-z0-9]{1,63}$`                                                                                             |
| `image`         | yourInitConatinerImageName:version which can pull from Dockerhub                                                                                    |
| `ports`         | List of ports to expose from initContainer                                                                                                          |
| `containerPort` | Number of port to expose on the pod's IP address. This must be a valid port number, `0 < x < 65536`                                                 |
| `workingDir`    | InitContainer's working directory. If not specified, the container runtime's default will be used, which might be configured in the container image |
| `command`       | The initContainer runs the command in Docker image.                                                                                                 |
| `volumeMounts`  | Pod volumes to mount into the container's filesystem                                                                                                |
| `name`          | Name of a volume, should be volume presents in `pod.spec.volumes`                                                                                   |
| `mountPath`     | Path within the container at which the volume should be mounted                                                                                     |

In above example, `taskRole: worker` and `taskRole: ps` have the same configuration except `containerPort`. 

For `initContainer`:

* It has name `initips`
* It use image [`zichengfang/k8s_initcontainer:update`](https://hub.docker.com/r/zichengfang/k8s_initcontainer/tags/), which can be built by [steps](#dockerimginitcontainer) above
* It mounts the shared Volume at `/configdir`
* In the command part:
    - First, export four environment variables, which will be used by `get_ips.sh`.  
    `worker_tasknum=1`, it has 1 `worker`; `ps_tasknum=1`, it has 1 `ps`; `worker_port=46866`, the port exposed by `worker`; `ps_port=46867`, the port exposed by `ps`.
    - Second, run `source ./get_ips.sh`, it can export `worker_hosts`, which is the list of `worker hostIP` *(exmaple:172.16.1.10:46866 and 172.16.1.10:46866,172.16.1.11:46866, if you have multiple workers)*;  
    it can also export `ps_hosts`, which is the list of `ps hostIP` *(exmaple:172.16.1.10:46867 and 172.16.1.10:46867,172.16.1.11:46867, if you have multiple ps)*
    - Last, the initContainer can write `$worker_hosts` and `$ps_hosts` into  `configfile` under shared Volume `/configdir`

For `container`:

* It has name `tf`
* It mounts the same shared Volume at `/configdir` as initContainer
* In the command part, it just run `cat /configdir/configfile`, which can read the configfile written by `initContainer`

### Configure a Pod to Use NFS Volume <a name="nfsvolume"></a>

In Tensorflow Framework, you will use some files and data. One way, you can use `NFS(Network File System)` to share your data, and an `nfs` volume allows NFS share to be mounted into your Pod. The content of an `nfs` volume are preserved even if a Pod is removed. An `nfs` volume can be prepopulated with data, and the data can be "handed off" between Pods, and one NFS can be mounted by multiple writers simultaneously.

*NOTE: you must have your own NFS server running with the share exported*

#### Set up a NFS Server <a name="nfsserver"></a>

This part introduces how to set up a NFS server on Linux (Ubuntu 16.04)

##### Step 1: Download and Install NFS Server

You need to install `nfs-kernel-server` package, which will allow you to share directories.

```console
$ sudo apt-get update
$ sudo apt-get install -y nfs-kernel-server
```

##### Step 2: Create the Share Directories

You need to create a directory to be shared and mounted.

```console
$ cd /home/t-zifang
$ mkdir nfs
```

NFS will translate any `root` operations on the client to the `nobody:nogroup` credentials as a security measure. Therefore, you need to change the directory ownership to match those credentials.

```console
$ sudo chown nobody:nogroup /home/t-zifang/nfs
```

This directory is now ready for export.

##### Step 3: Configure the NFS Exports

Open `/etc/exports` file in your test editor:

```console
$ sudo vi /etc/exports
```

Add the following content into the last line of exports configuration file:

```console
/home/t-zifang/nfs *(rw,sync,no_root_squash,subtree_check)
```

##### Step 4: Restart the NFS

```console
$ sudo /etc/init.d/rpcbind restart
$ sudo /etc/init.d/nfs-kernel-server restart
```

Now you finish setting up your NFS server, you can put directories and data under shared directory `/home/t-zifang/nfs`.

#### Mount on NFS Volume <a name="nfsmount"></a>

You have already set up your own NFS server, and you can configure a Pod to mount an NFS share by `nfs` volume.

Suppose on NFS server, you create a `nfstest.txt` under `/home/t-zifang/nfs`, and you insert some words:

```console
$ cd /home/t-zifang/nfs
$ echo Hello, NFS > nfstest.txt
```

You can config a Pod to use `nfs` volume to mount NFS server and read `nfstest.txt`.

Here is the example to use `nfs` volume:

```yaml
apiVersion: launcher.microsoft.com/v1
kind: Framework
metadata:
  name: nfsexample
spec:
  description: "tf example"
  executionType: Start
  retryPolicy:
    fancyRetryPolicy: false
    maxRetryCount: 3
  taskRoles:
    - name: worker
      taskNumber: 1
      frameworkAttemptCompletionPolicy:
        minFailedTaskCount: 1
        minSucceededTaskCount: -1
      task:
        retryPolicy:
          fancyRetryPolicy: false
          maxRetryCount: 3
        pod:
          spec:
            restartPolicy: Never
            containers:
              - name: nfstest
                image: "zichengfang/k8s_launcher:distributedtf"
                ports:
                - containerPort: 49866
                workingDir: /benchmarks/scripts/tf_cnn_benchmarks
                command: ["/bin/bash","-c","ls /mnt && cat /mnt/nfstest.txt"]
                volumeMounts:
                  - name: nfs-volume
                    mountPath: /mnt
            volumes:
              - name: nfs-volume
                nfs:
                  server: 10.169.4.12
                  path: /home/t-zifang/nfs
            hostNetwork: true
```

Under `taskRole.task.pod.spec.volumes`:

* You configure a volume name as `nfs-volume`
* You set the volume type as `nfs`
* You provide the NFS server IP `10.169.4.12`
* You provide the NFS shared directory path `/home/t-zifang/nfs`

Under `taskRole.task.pod.spec.containers.volumeMounts`:

* You configure a volumeMount name as `nfs-volume`, which must be same as the volume under `Volumes`
* You set a mountPath as `/mnt`, directories and files under `/home/t-zifang/nfs` can be accessed under `/mnt` in the container

In container `nfstest`, it runs `ls /mnt && cat /mnt/nfstest.txt`, and prints out:

```console
nfstext.txt
Hello, NFS
```

### Configure a Pod to Use hostPath Volume <a name="hostpathvolume"></a>

There is another way for Tensorflow Framework to use files and data. A `hostPath` volume mounts a file or directory from the host nodes's filesystem into your Pod.

#### Single Node using hostPath <a name="singlenodehp"></a>

You have only one node in your cluster. For example, you create a directory `/data/hostpathdir` on your host node, and you create a `hostpathtest.txt` under `/data/hostpathdir`, and you insert some words:

```console
$ cd /data
$ mkdir hostpathdir
$ cd hostpathdir 
$ echo Hello, HostPath > hostpathtest.txt
```

You can config a Pod to use `hostPath` volume to mount and read `hostpathtest.txt`.

Here is the example to use `hostPath` volume:

```yaml
apiVersion: launcher.microsoft.com/v1
kind: Framework
metadata:
  name: hostpathexample
spec:
  description: "tf example"
  executionType: Start
  retryPolicy:
    fancyRetryPolicy: false
    maxRetryCount: 3
  taskRoles:
    - name: worker
      taskNumber: 1
      frameworkAttemptCompletionPolicy:
        minFailedTaskCount: 1
        minSucceededTaskCount: -1
      task:
        retryPolicy:
          fancyRetryPolicy: false
          maxRetryCount: 3
        pod:
          spec:
            restartPolicy: Never
            containers:
              - name: hostpathstest
                image: "zichengfang/k8s_launcher:distributedtf"
                ports:
                - containerPort: 49866
                workingDir: /benchmarks/scripts/tf_cnn_benchmarks
                command: ["/bin/bash","-c","ls /mnt/hostpathdir && cat /mnt/hostpathdir/hostpathtest.txt"]
                volumeMounts:
                  - name: hostpath-volume
                    mountPath: /mnt
            volumes:
              - name: hostpath-volume
                hostPath:
                  path: /data
            hostNetwork: true
```

Under `taskRole.task.pod.spec.volumes`:

* You configure a volume name as `hostpath-volume`
* You set the volume type as `hostPath`
* You provide the directory path on node `/data`

Under `taskRole.task.pod.spec.containers.volumeMounts`:

* You configure a volumeMount name as `hostpath-volume`, which must be same as the volume under `Volumes`
* You set a mountPath as `/mnt`, directories and files under `/data` can be accessed under `/mnt` in the container

In container `hostpathstest`, it runs `ls /mnt/hostpathdir && cat /mnt/hostpathdir/hostpathtest.txt`, and prints out:

```console
hostpathtest.txt
Hello, HostPath
```

#### Multiple Nodes using hostPath <a name="multinodehp"></a>

You have multiple nodes on your cluster. Some nodes have file or data, the rest of them do not have. 

For example, you have 10 nodes, only node `10.151.40.241` and node `10.151.40.242` have `hostpathtest.txt` under their `/data/hostpathdir` directory. If you want to assign your Pod on node with `hostpathtest.txt`, you can configure [nodeAffinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/) for Pod to constrain a Pod to only be able to run on particular nodes.

*Nodes have labels, in test cluster, nodes have labels `kubernetes.io/hostname`, and values are `10.151.40.241`, `10.151.40.242`, `...`* 

Here is the example to use hostPath on cluster with multiple nodes and assign Pod to particular nodes:

```yaml
apiVersion: launcher.microsoft.com/v1
kind: Framework
metadata:
  name: hostpathexample
spec:
  description: "tf example"
  executionType: Start
  retryPolicy:
    fancyRetryPolicy: false
    maxRetryCount: 3
  taskRoles:
    - name: worker
      taskNumber: 1
      frameworkAttemptCompletionPolicy:
        minFailedTaskCount: 1
        minSucceededTaskCount: -1
      task:
        retryPolicy:
          fancyRetryPolicy: false
          maxRetryCount: 3
        pod:
          spec:
            affinity:
              nodeAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                  nodeSelectorTerms:
                  - matchExpressions:
                    - key: kubernetes.io/hostname
                      operator: In
                      values:
                      - 10.151.40.241
                      - 10.151.40.242
            restartPolicy: Never
            containers:
              - name: hostpathstest
                image: "zichengfang/k8s_launcher:distributedtf"
                ports:
                - containerPort: 49866
                workingDir: /benchmarks/scripts/tf_cnn_benchmarks
                command: ["/bin/bash","-c","ls /mnt/hostpathdir && cat /mnt/hostpathdir/hostpathtest.txt"]
                volumeMounts:
                  - name: hostpath-volume
                    mountPath: /mnt
            volumes:
              - name: hostpath-volume
                hostPath:
                  path: /data
            hostNetwork: true
```

Compared to configuration for single node, you need to add `affinity` under `taskRole.task.pod.spec`:

* You need to configure affinity type as `nodeAffinity`
* You need to configure `nodeAffinity` type as `requiredDuringSchedulingIgnoredDuringExecution`, which would "only run the Pod on nodes that matches the `matchExpression`"
* You need to configure `nodeSelectorTerms`
* You need to configure `matchExpressions` with `key: kubernetes.io/hostname`, `operator: In`, `values:` `- 10.151.40.241` `- 10.151.40.242`
* This node affinity rule says that the Pod can only be assigned to a node with a label whose key is `kubernetes.io/hostname` and whose value is either `10.151.40.241` or `10.151.40.242`

In container `hostpathstest`, it runs `ls /mnt/hostpathdir && cat /mnt/hostpathdir/hostpathtest.txt`, and prints out:

```console
hostpathtest.txt
Hello, HostPath
```

If we have 10 nodes on cluster and submit Framework without `nodeAffinity`, the Pod might be assigned to node other than `10.151.40.241` and `10.151.40.242`, such as `10.151.40.243`, you will not find and read `hostpathtest.txt`.

### Tensorflow Framework Examples <a name="tfexamples"></a>

In this part, we will show examples how to submit Tensorflow Frameworks by framework launcher on Kubernetes.

In following examples, we will deploy Tensorflow Framework using [Tensorflow CNN benchmarks](https://github.com/tensorflow/benchmarks):

* Deploy 1 `worker` and 1 `ps` 
* In application containers, it use image `zichengfang/k8s_launcher:distributedtf`, which can run `benchmarks` script
* In initContainers, it use image `zichengfang/k8s_initcontainer:update`, which can run `get_ips.sh` to get `worker` and `ps` hostIps list and export them as environment variables
* Tensorflow CNN Benchmarks need to use data, examples use `cifar10` data, which can be downloaded [here](https://www.cs.toronto.edu/~kriz/cifar-10-python.tar.gz)

In order to use `cifar10` data, we need to use `volume` to mount data. We will show how to use `nfs` volume and `hostPath` volume.

#### Tensorflow Framework using NFS <a name="tfnfs"></a>

Here is the example configuration to deploy Tensorflow Framework using NFS:

```yaml
apiVersion: launcher.microsoft.com/v1
kind: Framework
metadata:
  name: tfexample
spec:
  description: "tf example"
  executionType: Start
  retryPolicy:
    fancyRetryPolicy: false
    maxRetryCount: 3
  taskRoles:
    - name: worker
      taskNumber: 1
      frameworkAttemptCompletionPolicy:
        minFailedTaskCount: 1
        minSucceededTaskCount: -1
      task:
        retryPolicy:
          fancyRetryPolicy: false
          maxRetryCount: 3
        pod:
          spec:
            restartPolicy: Never
            containers:
              - name: tf
                # image: yourtensorflowjobimage:version
                image: zichengfang/k8s_launcher:distributedtf     
                ports:
                - containerPort: 49866
                workingDir: /benchmarks/scripts/tf_cnn_benchmarks
                command: ["/bin/bash","-c","export worker_hosts=$(head -n 1 /configdir/configfile) && export ps_hosts=$(tail -n 1 /configdir/configfile) && python tf_cnn_benchmarks.py --batch_size=32 --model=alexnet --local_parameter_device=cpu --variable_update=parameter_server --ps_hosts=$ps_hosts --worker_hosts=$worker_hosts --job_name=worker --task_index=$taskindex --data_name=cifar10 --data_dir=/mnt/cifar-10-batches-py/ --data_format=NHWC"]
                olumeMounts:
                  - name: config-volume
                    mountPath: /configdir
                  - name: data-volume
                    mountPath: /mnt
            initContainers:
              - name: initips
                # image: yourInitConatinerImageName:version
                image: zichengfang/k8s_initcontainer:update            
                ports:
                - containerPort: 49866
                workingDir: /
                command: ["/bin/bash","-c","export worker_tasknum=1 && export ps_tasknum=1 && export ps_port=46867 && export worker_port=46866 && source ./get_ips.sh && echo -e $worker_hosts'\n'$ps_hosts > /configdir/configfile"]
                volumeMounts:
                  - name: config-volume
                    mountPath: /configdir
            volumes:
              - name: config-volume
                emptyDir: {}
              - name: data-volume
                nfs:
                  server: 10.169.4.12
                  path: /home/t-zifang/nfs
            hostNetwork: true
    - name: ps
      taskNumber: 1
      frameworkAttemptCompletionPolicy:
        minFailedTaskCount: 1
        minSucceededTaskCount: -1
      task:
        retryPolicy:
          fancyRetryPolicy: false
          maxRetryCount: 3
        pod:
          spec:
            restartPolicy: Never
            containers:
              - name: tf
                # image: yourtensorflowjobimage:version
                image: zichengfang/k8s_launcher:distributedtf
                ports:
                - containerPort: 49867
                workingDir: /benchmarks/scripts/tf_cnn_benchmarks
                command: ["/bin/bash","-c","export worker_hosts=$(head -n 1 /configdir/configfile) && export ps_hosts=$(tail -n 1 /configdir/configfile) && python tf_cnn_benchmarks.py --batch_size=32 --model=alexnet --local_parameter_device=cpu --variable_update=parameter_server --ps_hosts=$ps_hosts --worker_hosts=$worker_hosts --job_name=ps --task_index=$taskindex --data_name=cifar10"]
                volumeMounts:
                  - name: config-volume
                    mountPath: /configdir
            initContainers:
              - name: initips
                # image: yourInitConatinerImageName:version
                image: zichengfang/k8s_initcontainer:update
                ports:
                - containerPort: 49867
                workingDir: /
                command: ["/bin/bash","-c","export worker_tasknum=1 && export ps_tasknum=1 && export ps_port=46867 && export worker_port=46866 && source ./get_ips.sh && echo -e $worker_hosts'\n'$ps_hosts > /configdir/configfile"]
                volumeMounts:
                  - name: config-volume
                    mountPath: /configdir
            volumes:
              - name: config-volume
                emptyDir: {}
            hostNetwork: true
```

For taskRole `ps`:

* Pod has 1 application container `tf`, 1 initContainer `initips`, 1 `emptyDir` type volume `config-volume`
* 
