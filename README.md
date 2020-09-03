# Investigating Cluster Operators On Openshift
As an OpenShift Technical Account Manager, helping Red Hat customers migrate from OpenShift 3 to OpenShift 4 has become a primary job function.  There are new terms, new objects, and generally an all around more automated experience in OpenShift 4.  The key part of this automation is the "Operator". I will assume that you have a passing knowledge of Operators but if you do not, a good primer on "Operators" can be found [here](https://www.redhat.com/en/topics/containers/what-is-a-kubernetes-operator). This blog post in particular will focus on cluster operators which make up the foundation of OpenShift; things like etcd, dns, authentication, etc.

If you're following along on your own cluster, you'll need a user with `cluster-admin` permissions and `oc` in your `$PATH`.  We'll also use Go templates so if you're not familiar with those, a self-led workshop is available [here](https://www.openshift.com/blog/customizing-oc-output-with-go-templates).  One last thing to note; we're using OpenShift 4.5 as the reference so there may be minor variations depending on your OpenShift version and installation method.  Now that we have the logistics covered, let's get started!

## Finding the Cluster Operators
The first thing we'll want to identify is what operators are part of the OpenShift platform. When you install an OpenShift 4 cluster, the bootstrap sets up the `ClusterOperator` custom resource definition (CRD) that describes a `ClusterOperator`.  Each of the `ClusterOperator` instances also provide one or more of their own CRDs for additional custom resources (CR) that they create and manage.  You can think of this like a family tree where the `cluster-bootsrap` is the parent of the `ClusterOperator` instances, and each `ClusterOperator` instance can in turn create its own children of a specific type (e.g. etcd-operator is an instance of `ClusterOperator` that creates and manages etcd instances). The tree structure is what allows the OpenShift cluster operators to be managed as a unit and the CRD allows the `kube-apiserver` to offer up a convenient API resource that we can query to get all of the `ClusterOperator`s as seen below.
```
$ oc get clusteroperators
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.5.4     True        False         False      42h
cloud-credential                           4.5.4     True        False         False      43h
cluster-autoscaler                         4.5.4     True        False         False      42h
config-operator                            4.5.4     True        False         False      42h
console                                    4.5.4     True        False         False      42h
...
```
Now that we know where to find the cluster operators, we'll want to start untangling what they're doing and other details that help us navigate the potential complexity.

## Untangling the Operator
One of the most common questions we hear at Red Hat for people who are just getting into Operators is: "How do I know what an Operator is doing?"

The best way to answer this question is to dive right into the YAML! We'll use the `machine-config` cluster operator as an example and start by looking at the `managedFields` section.
```
$ oc get co/machine-config -o yaml
...
  managedFields:
  - apiVersion: config.openshift.io/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:exclude.release.openshift.io/internal-openshift-hosted: {}
      f:spec: {}
      f:status:
        .: {}
        f:relatedObjects: {}
    manager: cluster-version-operator
    operation: Update
    time: "2020-09-02T16:48:05Z"
  - apiVersion: config.openshift.io/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:status:
        f:conditions: {}
        f:extension: {}
        f:versions: {}
    manager: machine-config-operator
    operation: Update
    time: "2020-09-02T16:56:09Z"
```
This looks a lot more complicated than it is.  Luckily, we can use OpenShift's built-in documentation to figure out what they mean! A snippet is below this explains what we're looking at.
```
$ oc explain clusteroperator.metadata.managedFields
KIND:     ClusterOperator
VERSION:  config.openshift.io/v1

RESOURCE: managedFields <[]Object>

DESCRIPTION:
     ManagedFields maps workflow-id and version to the set of fields that are managed by that workflow. This is mostly for internal housekeeping, and users typically shouldn't need to set or understand this field. A workflow can be the user's name, a controller's name, or the name of a specific apply path like "ci-cd". The set of fields is always in the version that the workflow used when modifying the object.

FIELDS:
   fieldsV1     <map[string]>
     FieldsV1 holds the first JSON version format as described in the "FieldsV1" type.

   manager      <string>
     Manager is an identifier of the workflow managing these fields.
```
To put it simply, `fieldsV1` holds the format, and `manager` holds the object that manages the format.  In the case of the `machine-config` cluster operator, that means that the fields listed in `managedFields[0]` are managed by the `cluster-version-operator` and the fields in `managedFields[1]` are managed by the `machine-config-operator`.

Let's continue with our YAML analysis with the `relatedObjects` section. What you see here is a list of resources which may or may not be custom resources, along with the API group they belong to and their name.  All of the items on this list are instantiated objects on the cluster so we can query any of them like a normal resource.
```
  relatedObjects:
  - group: ""
    name: openshift-machine-config-operator
    resource: namespaces
  - group: machineconfiguration.openshift.io
    name: master
    resource: machineconfigpools
  - group: machineconfiguration.openshift.io
    name: worker
    resource: machineconfigpools
  - group: machineconfiguration.openshift.io
    name: machine-config-controller
    resource: controllerconfigs
```
For an example, let's confirm that we see two `machineconfigpool` instances (as noted in `relatedObjects[1]` and `relatedObjects[2]`).
```
$ oc get machineconfigpool -o name
machineconfigpool.machineconfiguration.openshift.io/master
machineconfigpool.machineconfiguration.openshift.io/worker
```
One last note on this topic; you'll want to pay special attention to entries in `relatedObjects` that have a `namespace` field as that will help direct you to where that resource lives in the event that you need to do some troubleshooting.  In the case of the `machine-config` cluster operator none of the related objects have the namespace entry but if you check out the authentication cluster operator, you will see related objects that have a namespace noted.

Let's take a moment to recap what we've covered so far. We have:
* Picked a `ClusterOperator` to investigate.
* Run a command to get basic Operator info like it's version and state.
* Learned how to identify pieces of a `ClusterOperator` and what objects manage each piece of the cluster operator.
* Learned how to find objects related to a cluster operator.

## Leveraging Go Templates For A Holistic View
* Template for single operator view
* Template for all operators
* Templates for special cases

