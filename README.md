# Tanzu Application Engin / Tanzu HUB / Tanzu Application Platform
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
