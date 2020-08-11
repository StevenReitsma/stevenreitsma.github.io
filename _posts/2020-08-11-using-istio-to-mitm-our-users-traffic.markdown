---
layout: post
title:  "Using Istio to MITM our users' traffic"
date:   2020-08-11 17:00:00 +0200
image:  istio-mitm.jpg
tags:   Kubernetes Istio Security
---

I work as a consultant for various organizations in the Netherlands.
One of those organizations is the Dutch National Police, where I'm leading a team that's building a platform for data scientists to train machine learning models and easily deploy them to production.
The interesting thing about this organization is that there is a very strict policy on data access, and for good reason.

## GraphQL API
To comply with auditing requirements, data is wrapped by a GraphQL API that takes care of authentication and logging of data access.
Clients can then easily use this GraphQL API to query a myriad of different sources.
To keep all this data secure, we have to make sure this API adheres to some key security principles.

The API takes two sets of credentials: that of the person accessing it (the **user credentials**) and that of the system through which the user accesses the data (the **client credentials**).
Users each have their own user credentials, but they don't have direct access to the API.
Instead, they use various web or mobile interfaces, each of which has their own client credentials.
The data scientists and engineers are an exception here, as they **do** need direct access to the API.
But even they don't get local API access from their laptops, but rather use the data science platform we're building, based on Kubernetes, Istio and Kubeflow.
The user credentials are authenticated by a simple JWT that the data scientist retrieves when they log on to our identity provider.
The client credentials of our data science platform are an **access key** and a **secret key** that need to be kept from users --- otherwise they would be able to use these credentials to ingest data into uncontrolled systems.

## Security principles
The security mechanism of the data API works as follows:
- Every request is required to use TLS to guarantee **confidentiality** and **integrity**.
- The user's JWT needs to be included as a header for **authentication** and weak **non-repudiation**.
- The client's access key is set as a header, while the client's secret key is used to sign the entire request using an HMAC.
  This authenticates the system sending the API request. This is similar to how [AWS requires its S3 requests to be signed](https://docs.aws.amazon.com/AmazonS3/latest/API/sig-v4-authenticating-requests.html).
- The timestamp of the request needs to be added as an `X-Date` header to the request, to prevent **replay attacks**.
  The API backend checks whether the `X-Date` header doesn't deviate from the actual time by more than a few seconds.

## Data scientists
Data scientists love digging into data, training machine learning models and putting them into production.
But if there's one thing most data scientists agree on, it's that getting access to data isn't fun.
And with such a strict security regime, I understand the frustration of data scientists when they come across an API that requires them to compute an HMAC signature.
Next to that, the client access key and secret key should not be available to the data scientist, but only to the requesting system (our data science platform).
So we need to have some way of **transparently injecting the client credentials into the API request** to make it easier for data scientists to use the API, and to keep the platform's credentials safe.

## Cue Istio egress gateways
We're already using Istio in our cluster, so using an Istio egress gateway to accomplish our goal is a logical choice.
There are quite some tutorials on how to use Istio egress gateways for outbound traffic, but none of them show how to both **modify a request** and **keep the traffic encrypted in transit** at the same time.

The first step is to deploy the egress gateway.
You can do this by adding the following to your `IstioOperator` resource:

```yaml
egressGateways:
  - name: istio-egressgateway
    enabled: true
    namespace: egress
```

The next step is to create a `Gateway` resource that makes the gateway pod listen to requests on port 443 with the given TLS keypair:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: istio-egressgateway
  namespace: egress
spec:
  selector:
    istio: egressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      hosts:
        - api.data.local
      tls:
        serverCertificate: /etc/istio/egress-certs/tls.crt
        privateKey: /etc/istio/egress-certs/tls.key
        mode: SIMPLE
```
We create the keypair using cert-manager, but Istio does not yet support SDS for egress gateways.
It's a feature that [will be available in Istio 1.7](https://preliminary.istio.io/latest/docs/tasks/traffic-management/egress/egress-gateway-tls-origination-sds/).
For now we need to manually mount the TLS key and certificate into the egress gateway pod.
I'm not showing it here, but you can patch the istio-egressgateway `Deployment` to include a `Secret`-mount after deploying it.

We want to redirect all requests in the mesh intended for the data API to our egress gateway.
We can do this with a `VirtualService` resource:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: direct-data-source-through-egress-gateway
  namespace: egress
spec:
  hosts:
    - api.data.local
  gateways:
    - istio-egressgateway
    - mesh
  tls:
    - match:
        - gateways:
            - mesh
          port: 443
          sniHosts:
            - api.data.local
      route:
        - destination:
            host: istio-egressgateway.egress.svc.cluster.local
            subset: mitm
            port:
              number: 443
  http:
    - match:
        - gateways:
            - istio-egressgateway
          port: 443
      route:
        - destination:
            host: api.data.local
            port:
              number: 443
```
This does two things:
- The `spec.tls` part tells the sidecars in the mesh to redirect any request to `api.data.local:443` to `istio-egressgateway.egress.svc.cluster.local:443`
- The `spec.http` part tells Istio to route any request from the egress gateway to `api.data.local:443` to that same address.
  It's important to note that this latter part needs to be included in the `spec.http` part of the resource and not in the `spec.tls` part.
  This is because Istio initiates the request to the out-of-mesh service as an HTTP request, and only originates TLS after routing.

Because our data source API requires TLS, we need to originate TLS using a `DestinationRule` resource:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: originate-tls-for-datasource
  namespace: egress
spec:
  host: api.data.local
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
    portLevelSettings:
      - port:
          number: 443
        tls:
          mode: SIMPLE
```

We also need another `DestinationRule` to specify that traffic towards the egress gateway should be simple TLS (server-authentication only) instead of the default mTLS.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: egressgateway-for-datasource
  namespace: egress
spec:
  host: istio-egressgateway.egress.svc.cluster.local
  subsets:
    - name: mitm
      trafficPolicy:
        loadBalancer:
          simple: ROUND_ROBIN
        portLevelSettings:
          - port:
              number: 443
            tls:
              mode: SIMPLE
              sni: api.data.local
```

Lastly, we need to create a `ServiceEntry` to allow the egress gateway to access the out-of-mesh service (`api.data.local:443`), since we normally block all outbound requests using the `outboundTrafficPolicy: REGISTRY_ONLY` option of Istio.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: internal-datasource
  namespace: egress
spec:
  hosts:
    - api.data.local
  location: MESH_EXTERNAL
  ports:
    - number: 443
      name: https
      protocol: HTTPS
  resolution: DNS
```
Note that using wildcard hostnames is not supported for this use case.
The `ServiceEntry` resources need to have DNS resolution -- and thus FQDN hosts -- for the requests to be routed correctly.

## Injecting the credentials
We can now access the API through the egress gateway and traffic is encrypted at every transit.
Note that traffic is **not** end-to-end encrypted.
It's impossible to have end-to-end encryption and modify requests at the same time.

All that's left is to inject the necessary headers for authentication.
For that, we need an EnvoyFilter resource:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: inject-credentials-in-api-calls
  namespace: egress
spec:
  workloadSelector:
    labels:
      istio: egressgateway
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: GATEWAY
        listener:
          portNumber: 443
      patch:
        operation: INSERT_BEFORE
        value:
          typed_config:
            "@type": "type.googleapis.com/envoy.config.filter.http.lua.v2.Lua"
            inlineCode: |
              local hash = require("lib.hash")  -- this imports Egor-Skriptunoff/pure_lua_SHA

              local function ends_with(str, ending)
                 return ending == "" or str:sub(-#ending) == ending
              end

              function envoy_on_request(request_handle)
                if not ends_with(request_handle:headers():get(":authority"), "api.data.local") then
                  return
                end

                local method = request_handle:headers():get(":method")
                local host = request_handle:headers():get(":authority")
                local path = request_handle:headers():get(":path")
                local date = os.date("!%Y%m%dT%H%M%SZ")

                local signature_pretext = string.format("%s\n%s\n%s\n%s", method, host, path, date)

                local signature = hash.hmac(hash.sha256, secret_key, signature_pretext)
                local header = string.format("Version=V1, Credential=%s, Signature=%s", access_key, signature)

                request_handle:headers():add("X-Date", date)
                request_handle:headers():add("X-Authorization", header)
              end
          name: envoy.lua
```
I was really happy to see a pure-Lua SHA and HMAC module.
By importing it as a `ConfigMap` into the egress gateway pod I could easily use it as a module in the Envoy filter.
For me, this example really shows the awesome extensibility of Envoy.

Data scientists can now easily use the data API from their Jupyter Notebook, scheduled training job or streaming data applications without having to know the client credentials.
Of course they do still have to include their user credentials in the `Authorization` header.
We're currently working on a token exchange API to automatically insert this header as well.

```python
import requests

response = requests.post(
    "https://api.data.local/graphql",
    data={"query": "graphql-query-here"},
    headers={"Authorization": "Bearer user-jwt"}
)
```



<small>Note: this post was written with Istio 1.5.8 and Kubernetes 1.18.6.</small>
