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

## Turns out the mutation policy is a good to know, but the container has to run as non-root! 
I pulled a non root image from docker and placed it into my repo with out the pesky pull rate limit 

```
docker pull nginxinc/nginx-unprivileged
docker tag nginxinc/nginx-unprivileged gcr.io/boreal-rain-25xxx/nginx-unprivileged:latest
docker push gcr.io/boreal-rain-25xxx/nginx-unprivileged:latest
```

## here are all the steps I used to get the image to my on-prem cluster
The main commands are: 
```
tanzu login
cd orf-nginx-app-engine
tanzu app init
tanzu build config --build-plan-source-type=ucp --containerapp-registry gcr.io/boreal-rain-256712/{name}
tanzu deploy
```

The other commands are there for completnes sake to show what it actually takes

```
tanzu plugin install --group vmware-tanzu/app-developer
#
source tanzucli.src
export proj="AMER-East"
export sp="orfspace1"
export org="sa-tanzu-platform"
export cl="orfclustergroup"
export w=''
#export w='--wide'
export line="-----------------------------------------------------------------"
yes | tanzu context delete $org
# see POC_Tanzu_App_Engine for this file content
source ./tanzucli.src
tanzu login
#
git config --global credential.helper store
git config --global user.name "ogelbrich"
git config --global user.email "orf@gelbrich.com"
git config --global github.user ogelbrich
git config --global github.token ghp_re5sxxxxxxxxxxxxxxf0pLDuriZ2TpKdR
#
mkdir orf-nginx-app-engine
cd orf-nginx-app-engine/
#
echo "orf-nginx-app-engine" > README.md
git init
git commit -m "first commit"
git remote add origin https://github.com/ogelbric/orf-nginx-app-engine.git
curl -u ogelbrich:ghp_re5sxxxxxxxxxxxf0pLDuriZ2TpKdR https://api.github.com/user/repos -d '{"name":"orf-nginx-app-engine"}'
git push -u origin master
git commit -a
git push origin master
#
#
tanzu app init
? What is your app's name? orf-nginx-app-engine
? Which directory contains your app's source code? .
? Select container build type to use for this app: buildpacks

✓ Created tanzu.yml
✓ Recorded app configuration to .tanzu/config/orf-nginx-app-engine.yml

cp .tanzu/config/orf-nginx-app-engine.yml /tmp/orf-nginx-app-engine.yml
#
# The spec needs tobe changed (build needs to be replaces with image location)
#
#vi /tmp/orf-nginx-app-engine.yml #or# cp /tmp/orf-nginx-app-engine.yml .tanzu/config/orf-nginx-app-engine.yml

#sdiff .tanzu/config/orf-nginx-app-engine.yml /tmp/orf-nginx-app-engine.yml
#apiVersion: apps.tanzu.vmware.com/v1                            apiVersion: apps.tanzu.vmware.com/v1
#kind: ContainerApp                                              kind: ContainerApp
#metadata:                                                       metadata:
#  creationTimestamp: null                                         creationTimestamp: null
#  name: orf-nginx-app-engine                                      name: orf-nginx-app-engine
#spec:                                                           spec:
#  build:                                                      |   image: gcr.io/boreal-rain-256712/nginx-unprivileged.latest
#    buildpacks: {}                                            |   replicas: 4
#    path: ../..                                               <
#  ports:                                                          ports:
#  - name: main                                                    - name: main
#    port: 8080                                                      port: 8080
#
tanzu build config --build-plan-source-type=ucp --containerapp-registry gcr.io/boreal-rain-256712/{name}
#
docker login docker.io
tanzu project use $proj
tanzu space use $sp
git commit -a
git push origin master
#
tanzu deploy
#
```

Outcome: 

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/out1.png)

POD outcome: 

```
k get pods -n orfspace1-556bd95d54-f57r4 
NAME                                   READY   STATUS    RESTARTS   AGE
orf-nginx-app-engine-5d9f99845-7w452   2/2     Running   0          3m20s
orf-nginx-app-engine-5d9f99845-98cgj   2/2     Running   0          4m
orf-nginx-app-engine-5d9f99845-s64wm   2/2     Running   0          3m36s
orf-nginx-app-engine-5d9f99845-shdgw   2/2     Running   0          3m40s
```

