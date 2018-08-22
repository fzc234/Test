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
  - [Configure a Pod to Use NFS Volume](#nfsvolume)
    - [Set up a NFS Server](#nfsserver)
    - [Mount on NFS Volume](#nfsmount)
  - [Configure a Pod to Use hostPath Volume](#hostpathvolume)


## Simple Tensorflow Framework Example <a name="simpletf"></a>

To submit a Tensorflow Framework, you need to prepare a job configuration file and post it to APIServer.
`POST to http://{ApiServerAddress}/apis/launcher.microsoft.com/v1/namespaces/default/frameworks`

Here's one configuration file example:

```yaml
apiVersion: launcher.microsoft.com/v1
kind: Framework
metadata:
  name: tf-example
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
                image: "zichengfang/k8s_launcher:test4"
                ports:
                - containerPort: 49866
                workingDir: /benchmarks/scripts/tf_cnn_benchmarks
                command: ["/bin/bash","-c","export framework_name=tf-example && source ./get_ips.sh && python tf_cnn_benchmarks.py --batch_size=32 --model=alexnet --local_parameter_device=cpu --variable_update=parameter_server --ps_hosts=$ps_hosts:49867 --worker_hosts=$worker_hosts:49866 --job_name=worker --task_index=0 --data_name=cifar10 --data_dir=./data/cifar-10-batches-py/ --data_format=NHWC"]
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
                image: "zichengfang/k8s_launcher:test4"
                ports:
                - containerPort: 49867
                workingDir: /benchmarks/scripts/tf_cnn_benchmarks
                command: ["/bin/bash","-c","export framework_name=tf-example && source ./get_ips.sh && python tf_cnn_benchmarks.py --batch_size=32 --model=alexnet --local_parameter_device=cpu --variable_update=parameter_server --ps_hosts=$ps_hosts:49867 --worker_hosts=$worker_hosts:49866 --job_name=ps --task_index=0 --data_name=cifar10"]
```
