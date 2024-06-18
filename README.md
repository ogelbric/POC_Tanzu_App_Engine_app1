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
I pulled a non root image from docker and placed it into my repo without the pesky pull rate limit 

```
docker pull nginxinc/nginx-unprivileged
docker tag nginxinc/nginx-unprivileged gcr.io/boreal-rain-25xxx/nginx-unprivileged:latest
docker push gcr.io/boreal-rain-25xxx/nginx-unprivileged:latest
```

## Here are all the steps I used to get the image to my on-prem cluster
The main commands are: 
```
tanzu login
cd orf-nginx-app-engine
tanzu app init
This file: .tanzu/config/orf-nginx-app-engine.yml needs to be changed to image location!
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

In Tanzu Platform: 

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/out2.png)

## How to get ingress to my pod 

Step 1: Create a file k8sGatewayRoutes.yaml in {application deployment dir}/.tanzu/config (App name = orf-nginx-app-engine replace for your app name)

```
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: orf-nginx-app-engine-route
  annotations:
    healthcheck.gslb.tanzu.vmware.com/service: orf-nginx-app-engine
    healthcheck.gslb.tanzu.vmware.com/path: /
    healthcheck.gslb.tanzu.vmware.com/port: "80"
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: default-gateway
    sectionName: http-orf-nginx-app-engine #use https for TLS
  rules:
  - backendRefs:
    - group: ""
      kind: Service
      name: orf-nginx-app-engine
      port: 8080
      weight: 1
    matches:
    - path:
        type: PathPrefix
        value: /
```

Step 2: Create in AWS route 53 a sub domain (tanzu.gelbrich.com) / create NS record in gelbrich.com

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/nsdns1.png)

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/awssub1.png)

Step 3: Create a Credential in Tanzu HUB / Stack in AWS

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/cr1.png)

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/cr2.png)

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/cr3.png)

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/cr4.png)

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/cr5.png)

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/cr6.png)

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/cr7.png)

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/cr8.png)

Step 4: Create a custom network profile

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/net1.png)

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/net2.png)

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/net3.png)

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/net4.png)

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/net5.png)

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/net6.png)

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/net7.png)

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/net8.png)

Step 5: Update Space with new network policy

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/spnet1.png)

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/spnet2.png)

 Step 6: Re-load the application

 ```
cd orf-nginx-app-engine/
tanzu deploy
```

Outcome: 

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/outnet1.png)

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/outnet2.png)

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/outnet3.png)

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/outnet4.png)

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/outnet5.png)

![Version](https://github.com/ogelbric/POC_Tanzu_App_Engine_app1/blob/main/outnet6.png)

A few commands used to check things out

```
k get httproute -n orfspace1-5657bdb868-nb9sm 
k get httproute -n orfspace1-5657bdb868-nb9sm orf-nginx-app-engine-route   -o yaml
k get svc -A
k get gateway -A
k describe  gateway -n orfspace1-5657bdb868-nb9sm   default-gateway 
k get httproute -A
k describe httproute -n orfspace1-5657bdb868-nb9sm   orf-nginx-app-engine-route
```



# Docs Used
```
https://docs.vmware.com/en/VMware-Tanzu-Platform/services/create-manage-apps-tanzu-platform-k8s/how-to-build-and-deploy-from-source.html
https://docs.vmware.com/en/VMware-Tanzu-Platform/services/create-manage-apps-tanzu-platform-k8s/getting-started-create-app-envmt.html
```

