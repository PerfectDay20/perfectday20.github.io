+++
title = "The Tour of Exposing Kubernetes Dashboard Out of Cluster"

+++

> 💡 Note
>
> The newer or older version of Kubernetes dashboard may be easier to expose the services. This article is for version 7.11.1. 
>
> This article is from the viewpoint of a Kubernetes newbie, (many) mistakes may exist.

I have an All-In-One Ubuntu server running 24x7. One day when I was tweaking with Spark in a local mac, I wonder why not run it in the Ubuntu server? So I installed the YARN and HDFS on it. Running Spark on it was OK, but not too much fun. Then I remembered that Spark can also run on Kubernetes, and I have already deployed many docker containers on the server, so why not use this chance to learn the Kubernetes by running Spark on it?

That’s where this story started.

# Version info

- Kubernetes 1.32.3
- K3s v1.32.3+k3s1 (079ffa8d)
- Kubernetes Dashboard 7.11.1
- Envoy gateway api v1.3.2

# Install the Kubernetes

There are many ways to install a local development Kubernetes cluster, maybe too many to choose from. As a new comer I hesitated for a long time. After tried some distributions, I decided to use K3s.

My first try is Kind, which is listed first in the Kubernetes [getting started doc](https://kubernetes.io/docs/tasks/tools/), the name means “Kubernetes IN Docker”. Very easy to install, but hard to change the registry mirror. My network condition is very special that I need to use a registry mirror to access all kinds of the registries, so Kind is out.

My next try is Minikube, listed second in the doc. It uses docker to install the Kubernetes too. And the default resource settings can’t fully utilize my server, which is not preferred, out.

The next option is k3d, which is based on docker too. At that time I had already decided to run the Kubernetes cluster in bare metal and not in the docker, so k3d is out.

Then came the K3s, which fully fulfills my needs, can easily change the registry mirror, run in bare metal.

## Some changes to the K3s installation

### Change cluster version

By default K3s uses a [stable release channel](https://docs.k3s.io/upgrades/manual) to set up the cluster, which corresponding to Kubernetes 1.31 for now. But as I’m learning Kubernetes by reading its latest 1.32 doc, I need to install the version 1.32. This can be done to add `INSTALL_K3S_CHANNEL=latest` to the environment.

### Registry mirror

Create the registry mirror config file at `/etc/rancher/k3s/registries.yaml`:

```yaml
mirrors:
  docker.io:
    endpoint:
      - "https://example.foo.bar"
  registry.k8s.io:
    endpoint:
      - "https://example.foo.bar"
  gcr.io:
    endpoint:
      - "https://example.foo.bar"
  ghcr.io:
    endpoint:
      - "https://example.foo.bar"
  192.168.123.123:5000:
    endpoint:
      - "http://192.168.123.123:5000"
```

### Disable [AddOns](https://docs.k3s.io/installation/packaged-components)

K3s bring some very convenient AddOns. But since I want to learn Kubernetes in its default state, I need to disable some of them.

One is the local path provisioner, for automatically creating local volumes.

One is the traefik ingress controller, I’ll install a gateway controller instead.

There are [many ways to disable them](https://docs.k3s.io/installation/packaged-components#disabling-manifests):

- command line, like `--disable=traefik`
- config file at `/etc/rancher/k3s/config.yaml`
- `.skip` file

You’d better hard link the config files to other directory as a backup. Because in the future if you want to use the `/usr/local/bin/k3s-uninstall.sh` (created by installation) to uninstall the cluster, it will `rm -rf` the whole config dir.

# Install Kubernetes dashboard

By using helm, it’s very easy to install packages. Just follow the [official instructions](https://github.com/kubernetes/dashboard/blob/kubernetes-dashboard-7.11.1/README.md). 

But after installation, [the doc](https://github.com/kubernetes/dashboard/blob/kubernetes-dashboard-7.11.1/docs/user/accessing-dashboard/README.md) only shows some ways to expose the endpoint in the localhost, using port forward to expose the endpoint in localhost, or using `kubectl proxy` to expose through the API server. Neither way is graceful for long term usage.

In Kubernetes, an endpoint can be exposed by using:

- NodePort service. I’m not using this, as it’s not elegant.
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/). Not this either, because it’s feature-frozen and I don’t want to use a technology that is superseded.
- [Gateway](https://kubernetes.io/docs/concepts/services-networking/gateway/). That’s the way.

# Install gateway api controller

A gateway need a controller, and there are [TOO MANY controller implementations](https://gateway-api.sigs.k8s.io/implementations/) to choose from. Again, just like Kubernetes distributions, what a prosperous ecology! 

After viewing the [implementation status](https://gateway-api.sigs.k8s.io/implementations/v1.2/), there are some choices:

- Nginx, old and boring. [Old and boring is GOOD](https://boringtechnology.club/)! But I want to learn something new. So pass.
- Traefik, tried it before, not a good experience, so pass.
- Istio and Cillium. Implemented many features. But the installation is a little complicated, need another CLI tool? pass.
- Envoy. Yeah. I have saw it many times (even back to [2020](https://dropbox.tech/infrastructure/how-we-migrated-dropbox-from-nginx-to-envoy), maybe I’m the victim of the propaganda) Let’s try with it. (After using it, I found the docs are detailed and awesome, yet another reason to recommend it)

Still use helm to [install Envoy](https://gateway.envoyproxy.io/docs/tasks/quickstart/#installation), very easy.

Finish the quickstart, or at least create the [GatewayClass within](https://github.com/envoyproxy/gateway/blob/v1.3.2/examples/kubernetes/quickstart.yaml#L2) for future use. 

# Expose the dashboard

Then it’s time to configure the gateway and forward the traffic to the dashboard.

Dashboard use Kong proxy as a service, but only at port 443.

By using gateway, there are many ways to expose a service.

The current status is:

Browser —(downstream, http or https)—> Envoy gateway —(upstream, https)—> Kong

## Use HTTPRoute and BackendTLSPolicy

First I used gateway HTTP and HTTPS listeners with HTTPRoute to Kong’s 443 port, [HTTPRoute here only support Terminate TLS mode](https://gateway-api.sigs.k8s.io/guides/tls/). After the service and pods are created, I got this error because Envoy call Kong with HTTP:

```
400 Bad Request
The plain HTTP request was sent to HTTPS port
```

To change the Envoy’s call to HTTPS, I need to use a BackendTLSPolicy. Kong’s certificate is needed for creating the BackendTLSPolicy and I don’t find how to get it. Seems like the certificate is created somewhere in the installation process. Or I can create my own certificate and make Kong use it.

But I don’t want to learn Kong here, because it duplicates with Envoy as a proxy.

## Use TLS or TCP listener

Then I used the TLS listener with TLSRoute, and TCP listener with TCPRoute, both worked if the browser inited a HTTPS downstream request, because Envoy will not decrypt the package content and just passed it to the Kong.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: dashboard-gateway
  namespace: kubernetes-dashboard
spec:
  gatewayClassName: eg
  listeners:
    - name: tcp1
      protocol: TCP
      port: 80
    - name: tcp2
      protocol: TCP
      port: 443
---
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TCPRoute
metadata:
  name: tcproute
  namespace: kubernetes-dashboard
spec:
  parentRefs:
    - name: dashboard-gateway
  rules:
    - backendRefs:
        - group: ""
          kind: Service
          name: kubernetes-dashboard-kong-proxy
          port: 443
          weight: 1
```

https://example.local and https://example.local:80 both worked. But HTTPS with port 80? Awkward.

The returned certificate in browser is from Kong now.

In the future I want to use a sub path of the URL to access the dashboard, and use other paths for other services. Since TLS or TCP works in the lower network stack levels, the proxy is not handling the HTTP URL part, so it can’t rewrite the path.

So the solution goes back to the original one and I have to use the HTTPRoute.

## Use HTTPRoute and change the Kong’s config

If I have to use the HTTPRoute, then there is another choice: make Kong proxy serve http at port 80.

After fiddling with helm chart content for a while. I found dashboard helm chart [depends on Kong’s chart](https://github.com/kubernetes/dashboard/blob/kubernetes-dashboard-7.11.1/charts/kubernetes-dashboard/Chart.yaml#L46), then find some kong helm chart values.yaml. The default http proxy is [enabled](https://github.com/Kong/charts/blob/kong-2.46.0/charts/kong/values.yaml#L304), but the corresponding part in dashboard’s values.yaml is [disabled](https://github.com/kubernetes/dashboard/blob/kubernetes-dashboard-7.11.1/charts/kubernetes-dashboard/values.yaml#L399)! 

So thats the point!

Just copy the values.yaml, change the value, then use `helm upgrade` or just use the CLI to set the changed value.

HTTPS worked as expected. HTTP can see the login page, but can’t login. This may be caused by cookie secure settings? Because when clicked login, the browser didn't send correct cookies for subsequent requests in HTTP. It seems that the cookies are not set by Set-Cookie header, but by some JavaScript that reads the response, as the reponse contains the token being sent.

2025/04/12 Updated: The [doc](https://github.com/kubernetes/dashboard/blob/release/7.11.1/docs/user/access-control/creating-sample-user.md#accessing-dashboard) also has a note for this:
> Token login is ONLY allowed when the browser is accessing the UI over https. If your networking path to the UI is via http, the login will fail with an invalid token error.

But anyway, HTTPS works is all I need.

And there is still another way.

## Just drop kong proxy

[The Kong’s config file](https://github.com/kubernetes/dashboard/blob/kubernetes-dashboard-7.11.1/charts/kubernetes-dashboard/templates/config/gateway.yaml#L26) tells that Kong works by forward traffic to dashboard’s API, auth and web endpoint, so in theory I can disable the Kong proxy, and use Envoy to do it.

But to keep the change as small as possible, so in the future if the dashboard chart internal changes then I don’t need to dive too deep with it. So just keep it this way now. (From the [changelog](https://artifacthub.io/packages/helm/k8s-dashboard/kubernetes-dashboard#upgrading-an-existing-release-to-a-new-major-version), 6.x.x to 7.x.x added ingress-nginx-controller, then 7.x.x-alphaX to 7.x.x it was replaced by Kong, so there are always some architecture changes)

# Summary

- install k3s
- install Envoy gateway api
- change dashboard default settings
- create gateway, done

# Side note

Kubernetes is more advanced and opening (for extension) than YARN. There are even some good projects such as Helm to help installing packages in Kubernetes, which YARN doesn’t have(?). 

Although this comparison is not fair. YARN is created to support Hadoop and it successfully achieved the goals. But after truly using the Kubernetes, I can feel the robust of the Kubernetes and why [YARN is fading](https://www.reddit.com/r/bigdata/comments/c0a0ro/is_apache_hadoop_dying_is_it_already_dead/).