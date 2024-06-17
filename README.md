# Tanzu Application Engine / Tanzu HUB / Tanzu Platform
In this section I want to show how to add a simple nginx application from an existing POD into App Engine

The security settings in kubernetes ~1.26 / ~1.27 have changed and in order to just see can I get something up and running 
I am going to create a mutating policy for my namespaces to allow 

## Mutating Policy

I am looking for the following outcome (pod-security.kubernetes.io/enforce=privileged): 

```
k get ns orfns --show-labels
NAME    STATUS   AGE     LABELS
orfns   Active   6d18h   enforce=privileged,kubernetes.io/metadata.name=orfns,pod-security.kubernetes.io/enforce=privileged
```

Select Mutation

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/mut1.png)

Select Cluster Group and name for policy

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/mut2.png)

Select Label and namespace and (pod-security.kubernetes.io/enforce and privileged)

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/mut3.png)

Select nothing here

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/mut4.png)

Outcome

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/mut5.png)

