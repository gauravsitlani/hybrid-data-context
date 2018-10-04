# Hybrid Data Context

This tutorial runs on OpenShift, the easiest way to experience OpenShift is
using Minishift. For details about how to get setup Minishift on your local
machine I suggest the [Minishift Getting Started](https://docs.okd.io/latest/minishift/getting-started/installing.html) guide.

Startup Minishift

```
minishift start
```

Assuming this repository is your current working directory, we'll start by
creating a Ceph Nano service that can act as a private S3 compatable object
store for you Hybrid Data Context.

```
oc --as system:admin adm policy add-scc-to-user anyuid \
   system:serviceaccount:myproject:default
oc create -f ceph-rgw-keys.yml
oc create -f ceph-nano.yml
oc expose pod ceph-nano-0 --type=NodePort
```

Next, we'll take the incomplete openshift-spark build made available by the [radanalytics.io](https://radanalytics.io) community and use it create a custom openshift-spark build using a custom tarball that includes both Spark 2.3.2 with Hadoop 2.8.5. Then we'll instruct OpenShift to start building that image.

```
oc new-build --name=openshift-spark \
             --docker-image=radanalyticsio/openshift-spark-inc:latest \
             -e SPARK_URL=http://mmgaggle-bd.s3.amazonaws.com/spark-2.3.2-bin-hadoop-2.8.5.tgz \
             -e SPARK_MD5_URL=http://mmgaggle-bd.s3.amazonaws.com/spark-2.3.2-bin-hadoop-2.8.5.tgz.md5 \
             --binary
oc start-build openshift-spark
```

You can watch the image building process progress by tailing the buildconfig log.

```
oc logs -f buildconfig/openshift-spark
```

If you're tailing the log until the build completes, then you can press ctl-c to drop back to the shell. Next, we'll deploy a Jupyter notebook application into OpenShift using the openshift-spark image we built as the base, and provide a few environmental variables.

```
oc new-app -i myproject/base-notebook:latest \
           -e JUPYTER_NOTEBOOK_PASSWORD=developer \
           -e RGW_API_ENDPOINT=$(minishift openshift service ceph-nano-0 --url) \
           -e JUPYTER_NOTEBOOK_X_INCLUDE=https://raw.githubusercontent.com/mmgaggle/hybrid-data-context/master/hybrid-data-context.ipynb
```

Now we need to expose the OpenShift secrets to the Jupyter notebook application's environment.

```
oc env --from=secret/ceph-rgw-keys dc/base-notebook
```

Finally, we'll expose a route to the Jupyter notebook so we can access it from our browser.

```
oc expose svc/base-notebook
```



