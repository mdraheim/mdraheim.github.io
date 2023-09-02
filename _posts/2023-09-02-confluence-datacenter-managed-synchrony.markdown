---
layout: post
title:  "Confluence Datacenter with Managed Synchrony - Using Official Atlassian Chart and Image"
date:   2023-09-02 18:35:41 +0200
categories: atlassian
---
Doing the unstuck. It works! Atlassian marketing actually [recommends using managed synchrony for datacenter deployments](https://confluence.atlassian.com/doc/migrate-from-a-standalone-synchrony-cluster-to-managed-synchrony-958779077.html){:target="_blank"} for its *simpler setup, with less ongoing maintenance*. On the other hand, Atlassian helm chart devs [turned down feature reqests asking for support of managed synchrony](https://github.com/atlassian/data-center-helm-charts/issues/81){:target="_blank"} claiming issues with platform support and *2 separate JVM processes is not good practice in Kubernetes*.

Of course, my client was asking for exactly that, i.e. porting the existing Confluence server helm deployment driven by, ahem, my chart and, uhm, my image (this was all created when Atlassian was not providing either) to datacenter. That took me some time to figure out. And there I had full control over chart and image. Next step was to prove the concept with unmodified Atlassian chart and image. But let's take it step by step. First things first:

DISCLAIMER: Just because you can do it does not mean you should do it. There is no warranty for this setup being safe, let alone production ready. Any critical failures, data loss, screaming users, dead kittens are yours to keep. This page is meant as a proof of concept only. If in doubt, ask Atlassian support. Though we all can guess the most likely answer.

# Teaser

![confluence dc pods](/assets/images/20230902_01.png)

```
# confluence-1 container log
logClusterMembers Cluster now has 2 members: [[10.42.0.14]:5801, [10.42.0.15]:5801]
```
![confluence dc clustering](/assets/images/20230902_02.png)

![confluence dc clustering synchrony](/assets/images/20230902_03.png)

```
# cat logs/atlassian-synchrony.log
Members {size:2, ver:2} [
  Member [10.42.0.14]:5701 - 2f1234c5-c404-48f4-b1cf-b5d3addabbf4
  Member [10.42.0.15]:5701 - 15b9d0a3-1b6c-48bd-96ee-bb6375eb7052 this
]
```
![confluence pod synchrony](/assets/images/20230902_04.png)

# Test Setup

If you already have a kubernetes cluster and want to dive right into it, [check my chart overrides](https://gist.github.com/mdraheim/04812dd7fc5c97aeff2b83f128a204ac#file-confluence-dc-managed-synch-8-5-0-overrides-yaml){:target="_blank"} and happy helming.

I prefer a completely fresh playground for testing

* a single container Rancher "cluster" with istio, I might dedicate another post to this because it is just awesome
* clone of the [Atlassian helm charts](https://github.com/atlassian/data-center-helm-charts){:target="_blank"}
* if you are helm deploying from source checkout, copy the common "chart" into `confluence/charts/`.
* helm and kubectl binaries with kubeconfig fetched from Rancher

In the "cluster"

* create a namespace atl
* deploy a postgresql server
* optionally deploy istio gateway, virtualservice, destination rule
* create a hostpath pv and pvc to mount as shared-home into the confluence pods

# Overrides

Atlassian may not support the setup but with a few tricks in overrides one can bend the chart to do it. In summary, we need

* additional ports on the pod
* env override to disable the disablement of managed synchrony
* additional configmap to put synchrony properties into synchrony-args.properties

I will explain all the parts of the `diff -uw values.yaml overrides.yaml` from top to bottom.

Chart and image version here is 8.5.0

{% highlight diff %}
--- confluence/values.yaml	2023-08-29 21:08:36.319731865 +0200
+++ overrides.yaml	2023-09-02 20:13:33.285052094 +0200
@@ -60,7 +60,7 @@
   # will be auto-generated, otherwise the 'default' ServiceAccount will be used.
   # https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#use-the-default-service-account-to-access-the-api-server
   #
-  name:
+  name: confluence

   # -- For Docker images hosted in private registries, define the list of image pull
   # secrets that should be utilized by the created ServiceAccount
{% endhighlight %}

Name the serviceaccount. I basically chose "confluence" for everything.

{% highlight diff %}
@@ -145,7 +145,7 @@
    # - 'mssql'
   # https://atlassian.github.io/data-center-helm-charts/userguide/CONFIGURATION/#databasetype
   #
-  type:
+  type: postgresql

   # -- The jdbc URL of the database. If not specified, then it will need to be provided
   # via the browser during manual configuration post deployment. Example URLs include:
@@ -155,7 +155,7 @@
    # - 'jdbc:oracle:thin:@<dbhost>:1521:<SID>'
   # https://atlassian.github.io/data-center-helm-charts/userguide/CONFIGURATION/#databaseurl
   #
-  url:
+  url: 'jdbc:postgresql://postgresql:5432/confluence'

   # JDBC connection credentials
   #
{% endhighlight %}

Database. Could have just just the entrypoint envs and will do further down.

{% highlight diff %}
@@ -280,9 +280,9 @@
     # https://kubernetes.io/docs/concepts/storage/persistent-volumes/#static
     # https://atlassian.github.io/data-center-helm-charts/examples/storage/aws/SHARED_STORAGE/
     #
-    customVolume: {}
-    # persistentVolumeClaim:
-    #   claimName: "<pvc>"
+    customVolume:
+     persistentVolumeClaim:
+       claimName: "confluence"

     # -- Specifies the path in the Confluence container to which the shared-home volume will be
     # mounted.
{% endhighlight %}

Shared home pvc. In Rancher, just go to Storage and create a pv of hostpath type. Then deploy a pvc/confluence to atl namespace. Nevermind it being marked as RWO.

{% highlight diff %}
@@ -302,7 +302,7 @@
       # can write to it. This is a workaround for a K8s bug affecting NFS volumes:
       # https://github.com/kubernetes/examples/issues/260
       #
-      enabled: true
+      enabled: false

       # -- The path in the K8s initContainer where the shared-home volume will be mounted
       #
{% endhighlight %}

Disables the permission fixer. No idea why this would be necessary in a prod deployment.

{% highlight diff %}
@@ -451,12 +451,12 @@
   # this hostname will be routed by the Ingress Resource to the appropriate backend
   # Service.
   #
-  host:
+  host: 10-0-2-130.nip.io

   # -- The base path for the Ingress Resource. For example '/confluence'. Based on a
   # 'ingress.host' value of 'company.k8s.com' this would result in a URL of
   # 'company.k8s.com/confluence'. Default value is 'confluence.service.contextPath'
-  path:
+  path: /confluence

   # -- The custom annotations that should be applied to the Ingress Resource
   # when NOT using the K8s ingress-nginx controller.
@@ -466,7 +466,7 @@
   # -- Set to 'true' if browser communication with the application should be TLS
   # (HTTPS) enforced.
   #
-  https: true
+  https: false

   # -- The name of the K8s Secret that contains the TLS private key and corresponding
   # certificate. When utilised, TLS termination occurs at the ingress point where
@@ -489,7 +489,7 @@

     # -- The port on which the Confluence K8s Service will listen
     #
-    port: 80
+    port: 8090

     # -- The type of K8s service to use for Confluence
     #
@@ -519,7 +519,7 @@
     # -- The Tomcat context path that Confluence will use. The ATL_TOMCAT_CONTEXTPATH
     # will be set automatically.
     #
-    contextPath:
+    contextPath: /confluence

     # -- Additional annotations to apply to the Service
     #
{% endhighlight %}

Ingress and tomcat. With istio this is only relevant for tomcat server xml. I am kind of fond of nip.io or xip.io DNS reflectors to have "proper" base urls that do not point to localhost. This is optional, though. The service port change is optional and only here because my istio templates work this way.

{% highlight diff %}
@@ -532,7 +532,7 @@

   # -- Whether to apply security context to pod.
   #
-  securityContextEnabled: true
+  securityContextEnabled: false

   securityContext:

@@ -557,7 +557,7 @@
   # -- Boolean to define whether to set local home directory permissions on startup
   # of Confluence container. Set to 'false' to disable this behaviour.
   #
-  setPermissions: true
+  setPermissions: false

   # Port definitions
   #
{% endhighlight %}
Thins we do not need in testing.

{% highlight diff %}
@@ -569,7 +569,7 @@

     # -- The port on which the Confluence container listens for Hazelcast traffic
     #
-    hazelcast: 5701
+    hazelcast: 5801

   # Confluence licensing details
   #
{% endhighlight %}

This is important. Here I set confluence back to its default hazelcast port. Which means the upstream hazelcast default is free for synchrony to use. It would also be possible to do the reverse, like having confluence on 5701 and synchrony on 5801. But that makes the confusion complete.

{% highlight diff %}
@@ -595,7 +595,7 @@

     # -- Whether to apply the readinessProbe check to pod.
     #
-    enabled: true
+    enabled: false

     # -- The initial delay (in seconds) for the Confluence container readiness probe,
     # after which the probe will start running.
@@ -677,7 +677,7 @@

     # -- Set to 'true' if access logging should be enabled.
     #
-    enabled: true
+    enabled: false

     # -- The path within the Confluence container where the local-home volume should be
     # mounted in order to capture access logs.
{% endhighlight %}

More things we do not need for testing.

{% highlight diff %}
@@ -696,7 +696,7 @@
     # -- Set to 'true' if Data Center clustering should be enabled
     # This will automatically configure cluster peer discovery between cluster nodes.
     #
-    enabled: false
+    enabled: true

     # -- Set to 'true' if the K8s pod name should be used as the end-user-visible
     # name of the Data Center cluster node.
@@ -793,7 +793,7 @@
   # By default, confluence.cfg.xml is generated only once. Set `forceConfigUpdate` to true
   # to change this behavior.
   #
-  forceConfigUpdate: false
+  forceConfigUpdate: true

   # -- Specifies a list of additional arguments that can be passed to the Confluence JVM, e.g.
   # system properties.
{% endhighlight %}

Sure we want datacenter and yes we want the entrypoint to replace confluence.cfg.xml on disk. These overrides use emptyDir dir for local-home anyway.

{% highlight diff %}
@@ -890,14 +890,27 @@
   # container. See https://hub.docker.com/r/atlassian/confluence-server for
   # supported variables.
   #
-  additionalEnvironmentVariables: []
+  additionalEnvironmentVariables:
+    - name: JVM_SUPPORT_RECOMMENDED_ARGS
+      value: -Dconfluence.clusterNodeName.useHostname=true -Dhazelcast.kubernetes.service-port=5801 -Dsynchrony.proxy.enabled=true
+    - name: ATL_JDBC_USER
+      value: confluence
+    - name: ATL_JDBC_PASSWORD
+      value: confluence
+    - name: ATL_LICENSE_KEY
+      value: <timebomb license>

   # -- Defines any additional ports for the Confluence container.
   #
-  additionalPorts: []
-  #  - name: jmx
-  #    containerPort: 5555
-  #    protocol: TCP
+  additionalPorts:
+    - name: hazelsynchrony
+      containerPort: 5701
+      protocol: TCP
+    - name: hazelaleph
+      containerPort: 25500
+      protocol: TCP

   # -- Defines additional volumeClaimTemplates that should be applied to the Confluence pod.
   # Note that this will not create any corresponding volume mounts;
{% endhighlight %}

Here is where the fun starts. Atlassian devs actually hardcoded properties to disable synchrony in datacenter mode. See chart `_helpers.tpl` for JVM_SUPPORT_RECOMMENDED_ARGS. Luckily, additional envs defined here get templated into a position after the generated ones. So I can just override them with a duplicated env name. The first property is also in generated env, the second is kind of redundantly (because of the container env) telling the hazelcast plugin the service port which is important because it also applies to the **remote** ports hazelcast is scanning, the third property enables the manged synchrony-proxy.

Next is pod ports. This is especially important for istio because it will block any connection attempts to undeclared ports. I also added the aleph port, though I have no idea what that is used for. Note that those ports do not have to be listed in the svc manifest.

{% highlight diff %}
@@ -1375,15 +1388,15 @@
 # -- Create additional ConfigMaps with given names, keys and content. Ther Helm release name will be used as a prefix
 # for a ConfigMap name, fileName is used as subPath
 #
-additionalConfigMaps: []
-#  - name: extra-configmap
-#    keys:
-#      - fileName: hello.properties
-#        mountPath: /opt/atlassian/jira/atlassian-jira/WEB-INF/classes
-#        defaultMode:
-#        content: |
-#          foo=bar
-#          hello=world
+additionalConfigMaps:
+  - name: extra-configmap
+    keys:
+      - fileName: synchrony-args.properties
+        mountPath: /var/atlassian/application-data/confluence
+        defaultMode:
+        content: |
+          hazelcast.kubernetes.service-port=5701
+          cluster.interfaces=10.*.*.*
 #      - fileName: hello.xml
 #        mountPath: /opt/atlassian/jira/atlassian-jira/WEB-INF/classes
 #        content: |
{% endhighlight %}

This is the real secret. There is no way one can pass those porperties to synchrony through confluence properties. First property overrides the unfortunate env the chart will set to our chosen port 5801. Now synchrony will bind itself to local 5701 and will query remote hosts on the same port.

Second property is a workaround because confluence synchrony process builder insist on passing the property with value `0.0.0.0` to synchrony which, at least in my tests, then barfs out because of no interface matches this. I just set the value to `10.*.*.*` which should cover the pod's own ip.

# Helm It

Helm template with overrides, check output for obvious errors, then install. On first pod startup you are asked to import the sample data, set admin password, and create a FOO project. I suggest to scale down to 0 right after this and edit the confluence.cfg.xml on the shared-home pv. There set `confluence.cluster.authentication.enabled` to `false`. The setting will be picked up in next confluence pod. For unknown reasons, the cluster auth does not work in my istio clusters. It was the same for Jira where it had to be disabled by global java property.



