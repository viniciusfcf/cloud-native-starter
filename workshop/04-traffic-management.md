[home](README.md)
# Using traffic management in Kubernetes with Istio

****** **UNDER CONSTRUCTION** ******

The **“Cloud Native Starter”** is a sample polyglot microservices application with Java and Node.js on Kubernetes using Istio for traffic management, tracing, metrics, fault injection, fault tolerance, etc.

There are currently not many Istio examples available, the one most widely used is probably [Istio’s own “Bookinfo”](https://developer.ibm.com/solutions/container-orchestration-and-deployment/?cm_mmc=Search_Google-_-Developer_IBM+Developer-_-WW_EP-_-%2Bistio_b&cm_mmca1=000019RS&cm_mmca2=10004796&cm_mmca7=9041823&cm_mmca8=aud-396679157191:kwd-448983149697&cm_mmca9=_k_EAIaIQobChMIq_ynq8yi4gIVrDLTCh1T2g9AEAAYASAAEgIVAfD_BwE_k_&cm_mmca10=322762525080&cm_mmca11=b&gclid=EAIaIQobChMIq_ynq8yi4gIVrDLTCh1T2g9AEAAYASAAEgIVAfD_BwE) sample or the [Red Hat Istio tutorial](https://github.com/redhat-developer-demos/istio-tutorial). 

These tutorials and examples do focus on the request routing not as a part for a user-facing service behind the **Istio ingress**.

In this part we create a **new instance** and of a **new version** for the **web-api** microservice.

![git](images/traffic-new-architecture.gif)

We configure the routing to split the usage between our two instances and versions of our web-api microservice.

![gif](images/traffic-routing.gif)

## 1.1 Deployment definition

The interesting part is that these two versions of Web-API do exist as two different Kubernetes deployments and they run in parallel. That is definied in the [kubernetes-deployment-v1](web-api-java-jee/deployment/kubernetes-deployment-v1.yaml) and [kubernetes-deployment-v2](web-api-java-jee/deployment/kubernetes-deployment-v2.yaml).

Commonly, in Kubernetes we would replace v1 with v2. With **Istio** we can use two or more deployments of different versions of an app to do a **green/blue**, **A/B**, or [canary deployment](https://www.ibm.com/cloud/garage/tutorials/use-canary-testing-in-kubernetes-using-istio-toolchain) to test if v2 works as expected. We can slowly roll out our changes to a small subset of users before rolling it out to the entire infrastructure and making it available to everyone. 

 The **“version”** label: this is important for Istio to distinguish between the two deployments. 

| Version 1   |  Version 2    | 
|--- | --- |
| ![version 1](images/traffic-routing-deployment01.png)   |  ![version 1](images/traffic-routing-deployment02.png)    | 

## 1.2 Service definition

In Kubernetes we have one  [service definition](web-api-java-jee/deployment/kubernetes-service.yaml).

 The **selector** is only using the **“app”** label. Without Istio it will distribute traffic between the two deployments evenly. The port is named (“name: http”), because this is a [requirement](https://istio.io/docs/setup/kubernetes/prepare/requirements/) for Istio.

![service definition](images/traffic-routing-deployment03.png)

## 1.3 Istio gateway

Istio works with **envoy proxies** to **control** inbound and outbound traffic and to gather [telemetry data](https://en.wikipedia.org/wiki/Telemetry#Software) of a Kubernetes pod. The envoy is **injected as additional container** into a pod. The envoy **“sidecar”** allows to add Istio’s capabilities to an application without adding code or additional libraries to your application.

The image below is from the [Istio documentation](https://istio.io/docs/concepts/what-is-istio/) and shows the basic Istio architecture.

![Istio architecture](images/traffic-routing-deployment04.png)

The following image shows a simplified view on the given information for our situation. The Pod's do have **injected additional containers**.

![injected as additional containe](images/traffic-routing-deployment05.png)

Now we want to control the  route traffic (e.g. REST API calls). To control the traffic into a Kubernetes application a **Kubernetes Ingress** is required. 

With Istio, the equivalent is a **Istio Gateway** which allows it to manage and monitor incoming traffic. This gateway in turn uses the Istio ingress gateway which is a pod running in Kubernetes. This is the definition of an Istio gateway. The [Istio ingress.ymal](web-api-java-jee/deployment/istio-ingress.yaml).

This gateway listens on port **80** and answers to any request (“*”). The “hosts: *” should not be used in production, of course. 

![Istio gateway](images/traffic-routing-deployment06.png)

In the gif we can see a sample istio gateway instance the in Kubernetes.

![Istio gateway gif](images/traffic-routing-deployment07.gif)

## 1.4 Virtual Service

The second required Istio configuration object is a **“Virtual Service”** which overlays the Kubernetes service definition. The Web-API service in the example exposes 3 REST URIs. Two of them are used for API documentation (Swagger/Open API), they are **/openapi** and **/openapi/ui/** and are currently independent of the version of Web-API. 
The third URI is **/web-api/v1/getmultiple** and this is version-specific. 

Base on that we have following VirtualService definition:

![Virtual Service](images/traffic-routing-deployment08.png)

1. Pointer to the Ingress Gateway
2. URI that directly point to the Kubernetes service web-api listenting on port 9080 (without Istio)
3. URI that uses **“subset: v1”** of the service web-api which we haven’t defined yet, this is Istio specific
4. Root **/** is pointing to port **80** of the web-app service which is different from web-api! It is the service that provides the Vue app to the browser.

## 1.5 Destination rule

To control the traffic we need to define a DestinationRule, this is  Istio specific. 

In the image below we can see, the subset v1 is selecting pods that belong to web-api and have a selector label of “version: v1” which is the deployment “web-api-v1”.

![Destination rule](images/traffic-routing-deployment09.png)

With this Istio rule set in place all incoming traffic will go to version 1 of the Web-API. 

In the following image you can see all the traffic is routed to version 1.

![Sample traffic](images/traffic-routing-deployment10.png)

## 1.6 Traffic distribution

We can change the VirtualService to distribute incoming traffic, e.g. 80% should go to version 1, 20% should go to version 2:






## 1.7 Lab - Traffic Routing

In order to demonstrate traffic routing you can run the following commands. 20 % of the web-api API request to read articles will now return 10 instead of 5 articles which is version 2. 80 % of the requests are still showing only 5 articles which is version 1. 

```
$ cd $PROJECT_HOME
$ iks-scripts/check-prerequisites.sh
$ iks-scripts/delete-all.sh
$ iks-scripts/deploy-articles-java-jee.sh
$ iks-scripts/deploy-authors-nodejs.sh
$ iks-scripts/deploy-web-app-vuejs.sh
$ iks-scripts/deploy-web-api-java-jee.sh
$ iks-scripts/deploy-web-api-java-jee-v2.sh
$ iks-scripts/deploy-istio-ingress-v1-v2.sh
$ iks-scripts/show-urls.sh
```

Now, we've finished the **Using traffic management in Kubernetes**.
Let's get started with the [Lab - Resiliency](05-resiliency.md).

---

Resources:

* ['Managing Microservices Traffic with Istio'](https://haralduebele.blog/2019/03/11/managing-microservices-traffic-with-istio/)
* ['Demo: Traffic Routing'](../documentation/DemoTrafficRouting.md)