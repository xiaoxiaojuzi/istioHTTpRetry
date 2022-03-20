# istioHTTpRetry
# istio HTTPRetry for external call
## 1. istio HTTPRetry for internal call
Describes the retry policy to use when a HTTP request fails. For [example](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPRetry), the following rule sets the maximum number of retries to 3 when calling ratings:v1 service, with a 2s timeout per retry attempt. A retry will be attempted if there is a connect-failure, refused_stream or when the upstream server responds with Service Unavailable(503).
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings.prod.svc.cluster.local
  http:
  - route:
    - destination:
        host: ratings.prod.svc.cluster.local
        subset: v1
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: connect-failure,refused-stream,503
```

[note:](https://istio.io/latest/docs/concepts/traffic-management/#destination)  the `destination`’s host must be a real destination that exists in Istio’s service registry or Envoy won’t know where to send traffic to it. This can be a mesh service with proxies or a non-mesh service added using a service entry. 
## 2. istio HTTPRetry for external call
### 2.1 Service entries
You use a service entry to add an [entry](https://istio.io/latest/docs/concepts/traffic-management/#service-entries) to the service registry that Istio maintains internally. After you add the service entry, the Envoy proxies can send traffic to the service as if it was a service in your mesh. Configuring service entries allows you to manage traffic for services running outside of the mesh, including retry:

#### Add service entry
add service entry for `http://httpstat.us`
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: httpstat.us
spec:
  hosts:
  - httpstat.us
  ports:
  - number: 80
    name: http
    protocol: HTTP
  location: MESH_EXTERNAL
  resolution: DNS
```
### 2.2 Add VirtualService for the service entry
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpstat.us
spec:
  hosts:
  - httpstat.us
  http:
  - route:
    - destination:
        host: httpstat.us
    retries:
      attempts: 3
      perTryTimeout: 20s
      retryOn: connect-failure,refused-stream,503
```
### 2.3 Test 
In the pod `ratings-v1-b6994bb9-57lqg`, send a http request: 
1. Into the pod `ratings-v1-b6994bb9-57lqg`:
`kubectl exec -it ratings-v1-b6994bb9-57lqg /bin/bash`
2. To call the external api `http://httpstat.us/503` that response is 503 from server, run the command `curl http://httpstat.us/503 -I` in pod. 

we can test whether the retry police has trigger by using charles proxy. 
![Alt text](./1647787002019.png)

# Next to spike
To retry `aws sqs send message` call, does we need other config.
