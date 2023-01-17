# GRPC guide for APICast

> Forked: https://github.com/3scale-demos/apicast-demo-apps

This explain how to setup GRPC calls on APICast server. 


## Pre-req (Optional)

> All docker images are located in quay.io/3scale organization, but if you want to build your own, here are the instructions:

GRPC client and server

```shell
cd source
export TARGET=quay.io/3scale/grpc-utils:golang
docker build -t ${TARGET} -f client.dockerfile .
docker push ${TARGET}
```

Certificates:

The certificates are embedded into the yaml files, but if you want to change the
values this is how are generated.

Root CA certificate:
```shell
openssl genrsa -out rootCA.key 2048
openssl req -batch -new -x509 -nodes -key rootCA.key -sha256 -days 3650 -out rootCA.pem
```

Domain certificates for APICast

```shell
export DOMAIN=apicast-service
openssl req -subj "/CN=${DOMAIN}"  -newkey rsa:4096 -nodes \
    -sha256 \
    -days 3650 \
    -keyout ${DOMAIN}.key \
    -out ${DOMAIN}.csr
openssl x509 -req -in ${DOMAIN}.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out ${DOMAIN}.crt -days 3650 -sha256
```

Domain certificates for GRPC endpoint:

```shell
export DOMAIN=grpc-service
openssl req -subj "/CN=${DOMAIN}"  -newkey rsa:4096 -nodes \
    -sha256 \
    -days 3650 \
    -keyout ${DOMAIN}.key \
    -out ${DOMAIN}.csr
openssl x509 -req -in ${DOMAIN}.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out ${DOMAIN}.crt -days 3650 -sha256
```



## Installation

Openshift is needed to run this example:

```shell
# create a project to deploy the demo
❯ oc new-project grpc-demo

# create the resources
❯ oc kustomize k8s | oc apply -f -
```

> artifacts creates

```shell
❯ oc get pod,svc,route
NAME                     READY   STATUS      RESTARTS   AGE
pod/apicast-1-deploy     0/1     Completed   0          110s
pod/apicast-1-x4nx4      1/1     Running     0          107s
pod/apicast-api-server   1/1     Running     0          110s
pod/client               1/1     Running     0          109s
pod/grpc-endpoint        1/1     Running     0          109s

NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
service/apicast-service   ClusterIP   10.152.227.71    <none>        8080/TCP,8043/TCP   112s
service/grpc-service      ClusterIP   10.152.139.174   <none>        443/TCP             111s
service/server-service    ClusterIP   10.152.158.76    <none>        80/TCP              111s
```



## Testing 

### Address using Service directly

1. Client directly call the grpc service. Plaintext connection.

```shell
oc exec -ti client --  /go/bin/client --address grpc-service:443 --name "Rfelix"
```

> <u>output</u>:
>
> ```shell
> 	- Address: 172.19.0.3:8043
> 	- Name: world
> 	- User_key: test
> 2023/01/17 13:39:36 Greeting: Hello Rfelix
> ```

2. Client using APICast.

```shell
oc exec -ti client --  /go/bin/client --address apicast-service:8043 --name "Bob"
```

> <u>output</u>:
>
> ```shell
> 	- Address: 172.19.0.3:8043
> 	- Name: world
> 	- User_key: test
> 2023/01/17 13:41:01 Greeting: Hello Bob
> ```
>
> <u>output</u> of logs: `oc logs -f dc/apicast --tail=1`
>
> ```shell
> 2023/01/17 13:41:01 [debug] 26#26: *371 executor.lua:26: ssl_certificate(): executor phase: ssl_certificate
> 2023/01/17 13:41:01 [debug] 26#26: *371 policy_chain.lua:213: ssl_certificate(): policy chain execute phase: ssl_certificate, policy: Load Configuration, i: 1
> 2023/01/17 13:41:01 [debug] 26#26: *371 policy_chain.lua:213: ssl_certificate(): policy chain execute phase: ssl_certificate, policy: Find Service Policy, i: 2
> 2023/01/17 13:41:01 [debug] 26#26: *371 policy_chain.lua:213: ssl_certificate(): policy chain execute phase: ssl_certificate, policy: Local Policy Chain, i: 3
> 2023/01/17 13:41:01 [debug] 26#26: *371 policy_chain.lua:213: policy chain execute phase: ssl_certificate, policy: tls, i: 1
> 2023/01/17 13:41:01 [debug] 26#26: *370 executor.lua:26: rewrite(): executor phase: rewrite, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: rewrite(): policy chain execute phase: rewrite, policy: Load Configuration, i: 1, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: rewrite(): policy chain execute phase: rewrite, policy: Find Service Policy, i: 2, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 find_service.lua:43: Using service id=200, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: rewrite(): policy chain execute phase: rewrite, policy: Local Policy Chain, i: 3, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 cache_handler.lua:23: new(): backend cache handler: strict, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: rewrite(): policy chain execute phase: rewrite, policy: tls, i: 1, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: rewrite(): policy chain execute phase: rewrite, policy: grpc, i: 2, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: rewrite(): policy chain execute phase: rewrite, policy: APIcast, i: 3, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 service.lua:236: get_usage(): [mapping] service 200 has 2 rules, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [warn] 26#26: *370 a client request body is buffered to a temporary file /tmp/lua_zurVbm/client_body_temp/0000000009, client: 10.148.16.57, server: _, request: "POST /helloworld.Greeter/SayHello HTTP/2.0", host: "apicast-service:8043"
> 2023/01/17 13:41:01 [notice] 26#26: *370 service.lua:184: get_request_params(): Error while getting post args: request body in temp file not supported, requestID=d763231c5e856e2adc239775a8f6d56c, client: 10.148.16.57, server: _, request: "POST /helloworld.Greeter/SayHello HTTP/2.0", host: "apicast-service:8043"
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: rewrite(): policy chain execute phase: rewrite, policy: Metrics, i: 4, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 executor.lua:26: access(): executor phase: access, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: access(): policy chain execute phase: access, policy: Load Configuration, i: 1, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: access(): policy chain execute phase: access, policy: Find Service Policy, i: 2, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: access(): policy chain execute phase: access, policy: Local Policy Chain, i: 3, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: access, policy: tls, i: 1, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: access, policy: grpc, i: 2, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: access, policy: APIcast, i: 3, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [info] 26#26: *370 proxy.lua:82: output_debug_headers(): usage: usage%5Bhits%5D=1 credentials: user_key=test, requestID=d763231c5e856e2adc239775a8f6d56c, client: 10.148.16.57, server: _, request: "POST /helloworld.Greeter/SayHello HTTP/2.0", host: "apicast-service:8043"
> 2023/01/17 13:41:01 [debug] 26#26: *370 proxy.lua:133: apicast cache hit key: 200:test:usage%5Bhits%5D=1, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: access(): policy chain execute phase: access, policy: Metrics, i: 4, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 executor.lua:26: content(): executor phase: content, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: content(): policy chain execute phase: content, policy: Load Configuration, i: 1, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: content(): policy chain execute phase: content, policy: Find Service Policy, i: 2, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: content(): policy chain execute phase: content, policy: Local Policy Chain, i: 3, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: content, policy: tls, i: 1, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: content, policy: grpc, i: 2, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: content, policy: APIcast, i: 3, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 upstream.lua:209: set_keepalive_key(): keepalive key for https connection set to: 'grpc-service::200', requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 resolver.lua:196: new(): resolver search domains:  grpc-demo.svc.cluster.local svc.cluster.local cluster.local k8sstgbb103.desenv.bb.com.br dispositivos.bb.com.br grpc-demo.svc.cluster.local svc.cluster.local cluster.local k8sstgbb103.desenv.bb.com.br dispositivos.bb.com.br, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 resolver.lua:350: lookup(): resolver query: grpc-service, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 resolver.lua:321: search_dns(): resolver query: grpc-service search:  query: grpc-service., requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 cache.lua:122: fetch_answers(): resolver cache miss grpc-service., requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 cache.lua:188: get(): resolver cache miss: grpc-service., requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 dns_client.lua:50: init_resolvers(): initializing 2 nameservers, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 dns_client.lua:63: init_resolvers(): nameserver 10.152.0.10:53 initialized, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 dns_client.lua:63: init_resolvers(): nameserver 10.152.0.10:53 initialized, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 dns_client.lua:75: query(): resolver query: grpc-service. nameserver: 10.152.0.10:53, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 dns_client.lua:75: query(): resolver query: grpc-service. nameserver: 10.152.0.10:53, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 resolver.lua:321: search_dns(): resolver query: grpc-service search: grpc-demo.svc.cluster.local query: grpc-service.grpc-demo.svc.cluster.local, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 cache.lua:122: fetch_answers(): resolver cache miss grpc-service.grpc-demo.svc.cluster.local, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 cache.lua:188: get(): resolver cache miss: grpc-service.grpc-demo.svc.cluster.local, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 dns_client.lua:75: query(): resolver query: grpc-service.grpc-demo.svc.cluster.local nameserver: 10.152.0.10:53, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 cache.lua:74: store(): resolver cache write grpc-service.grpc-demo.svc.cluster.local with TLL 5, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 resolver.lua:370: lookup(): resolver query: grpc-service finished with 1 answers, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 resolver.lua:410: get_servers(): query for grpc-service finished with 1 answers, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: balancer(): policy chain execute phase: balancer, policy: Load Configuration, i: 1, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: balancer(): policy chain execute phase: balancer, policy: Find Service Policy, i: 2, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: balancer(): policy chain execute phase: balancer, policy: Local Policy Chain, i: 3, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: balancer, policy: tls, i: 1, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: balancer, policy: grpc, i: 2, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [info] 26#26: *370 balancer.lua:108: set_current_peer(): balancer set peer 10.152.248.130:443 ok: true err: nil, requestID=d763231c5e856e2adc239775a8f6d56c while connecting to upstream, client: 10.148.16.57, server: _, request: "POST /helloworld.Greeter/SayHello HTTP/2.0", host: "grpc-service"
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: balancer, policy: APIcast, i: 3, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: balancer(): policy chain execute phase: balancer, policy: Metrics, i: 4, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 Connection Handler started for request:'00005596259EF540' and ctx:'0000559625999510'
> 2023/01/17 13:41:01 [debug] 26#26: *370 SetProxyCert: no certificate was set
> 2023/01/17 13:41:01 [debug] 26#26: *370 SetProxyCertKey: certificate key was not found
> 2023/01/17 13:41:01 [debug] 26#26: *370 executor.lua:26: header_filter(): executor phase: header_filter, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: header_filter(): policy chain execute phase: header_filter, policy: Load Configuration, i: 1, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: header_filter(): policy chain execute phase: header_filter, policy: Find Service Policy, i: 2, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: header_filter(): policy chain execute phase: header_filter, policy: Local Policy Chain, i: 3, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: header_filter, policy: tls, i: 1, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: header_filter, policy: grpc, i: 2, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: header_filter, policy: APIcast, i: 3, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: header_filter(): policy chain execute phase: header_filter, policy: Metrics, i: 4, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 executor.lua:26: body_filter(): executor phase: body_filter, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: body_filter(): policy chain execute phase: body_filter, policy: Load Configuration, i: 1, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: body_filter(): policy chain execute phase: body_filter, policy: Find Service Policy, i: 2, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: body_filter(): policy chain execute phase: body_filter, policy: Local Policy Chain, i: 3, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: body_filter, policy: tls, i: 1, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: body_filter, policy: grpc, i: 2, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: body_filter, policy: APIcast, i: 3, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: body_filter(): policy chain execute phase: body_filter, policy: Metrics, i: 4, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 executor.lua:26: body_filter(): executor phase: body_filter, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: body_filter(): policy chain execute phase: body_filter, policy: Load Configuration, i: 1, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: body_filter(): policy chain execute phase: body_filter, policy: Find Service Policy, i: 2, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: body_filter(): policy chain execute phase: body_filter, policy: Local Policy Chain, i: 3, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: body_filter, policy: tls, i: 1, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: body_filter, policy: grpc, i: 2, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: body_filter, policy: APIcast, i: 3, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: body_filter(): policy chain execute phase: body_filter, policy: Metrics, i: 4, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 executor.lua:26: post_action(): executor phase: post_action, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: post_action(): policy chain execute phase: post_action, policy: Load Configuration, i: 1, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: post_action(): policy chain execute phase: post_action, policy: Find Service Policy, i: 2, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: post_action(): policy chain execute phase: post_action, policy: Local Policy Chain, i: 3, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: post_action, policy: tls, i: 1, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: post_action, policy: grpc, i: 2, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: post_action, policy: APIcast, i: 3, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [info] 26#26: *370 proxy.lua:339: [async] reporting to backend asynchronously, cached_key: 200:test:usage%5Bhits%5D=1, requestID=d763231c5e856e2adc239775a8f6d56c while sending to client, client: 10.148.16.57, server: _, request: "POST /helloworld.Greeter/SayHello HTTP/2.0", upstream: "grpcs://10.152.248.130:443", host: "grpc-service"
> 2023/01/17 13:41:01 [debug] 26#26: *370 resolver.lua:350: lookup(): resolver query: server-service, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 resolver.lua:321: search_dns(): resolver query: server-service search:  query: server-service., requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 cache.lua:122: fetch_answers(): resolver cache miss server-service., requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 cache.lua:188: get(): resolver cache miss: server-service., requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 dns_client.lua:75: query(): resolver query: server-service. nameserver: 10.152.0.10:53, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [info] 26#26: *370 client prematurely closed connection while processing HTTP/2 connection, client: 10.148.16.57, server: 0.0.0.0:8043
> 2023/01/17 13:41:01 [debug] 26#26: *370 dns_client.lua:75: query(): resolver query: server-service. nameserver: 10.152.0.10:53, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 resolver.lua:321: search_dns(): resolver query: server-service search: grpc-demo.svc.cluster.local query: server-service.grpc-demo.svc.cluster.local, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 cache.lua:122: fetch_answers(): resolver cache miss server-service.grpc-demo.svc.cluster.local, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 cache.lua:188: get(): resolver cache miss: server-service.grpc-demo.svc.cluster.local, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 dns_client.lua:75: query(): resolver query: server-service.grpc-demo.svc.cluster.local nameserver: 10.152.0.10:53, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 cache.lua:74: store(): resolver cache write server-service.grpc-demo.svc.cluster.local with TLL 5, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 resolver.lua:370: lookup(): resolver query: server-service finished with 1 answers, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 resolver.lua:410: get_servers(): query for server-service finished with 1 answers, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 resolver.lua:350: lookup(): resolver query: 10.152.194.110, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 resolver.lua:355: lookup(): host is ip address: 10.152.194.110, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 resolver.lua:370: lookup(): resolver query: 10.152.194.110 finished with 1 answers, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 resolver.lua:410: get_servers(): query for 10.152.194.110 finished with 1 answers, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 http.lua:50: connect(): connected to  ip:10.152.194.110 host: 10.152.194.110 port: 80 ok: 1 err: nil, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 proxy.lua:26: new(): connection to server-service:80 established, reused times: 0, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [info] 26#26: *370 backend_client.lua:133: call_backend_transaction(): backend client uri: http://server-service/transactions/authrep.xml?service_token=token-value&service_id=200&usage%5Bhits%5D=1&user_key=test ok: true status: 200 body: "hello world"
>  error: nil, requestID=d763231c5e856e2adc239775a8f6d56c while sending to client, client: 10.148.16.57, server: _, request: "POST /helloworld.Greeter/SayHello HTTP/2.0", upstream: "grpcs://10.152.248.130:443", host: "grpc-service"
> 2023/01/17 13:41:01 [debug] 26#26: *370 proxy.lua:381: handle_backend_response(): [backend] response status: 200 body: "hello world"
> , requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: post_action(): policy chain execute phase: post_action, policy: Metrics, i: 4, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 executor.lua:26: log(): executor phase: log, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: log(): policy chain execute phase: log, policy: Load Configuration, i: 1, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: log(): policy chain execute phase: log, policy: Find Service Policy, i: 2, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: log(): policy chain execute phase: log, policy: Local Policy Chain, i: 3, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: log, policy: tls, i: 1, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: log, policy: grpc, i: 2, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: policy chain execute phase: log, policy: APIcast, i: 3, requestID=d763231c5e856e2adc239775a8f6d56c
> 2023/01/17 13:41:01 [debug] 26#26: *370 policy_chain.lua:213: log(): policy chain execute phase: log, policy: Metrics, i: 4, requestID=d763231c5e856e2adc239775a8f6d56c
> [17/Jan/2023:13:41:01 +0000] apicast-service:8043 10.148.16.57:32898 "POST /helloworld.Greeter/SayHello HTTP/2.0" 200 16 (0.100) 0
> ```

3. The current user_key is "test", but can be changed with the following command:

```shell
oc exec -ti client --  /go/bin/client --address apicast-service:8043 --name "Bob" --user_key="${MY_SECRET_KEY}"
```

> Client **help** is the following, the connection is TLS but without validate the certificates.

```shell
  -address string
        address to connect to (default "172.19.0.3:8043")
  -name string
        Name to say hello (default "world")
  -user_key string
        user_key to use it (default "test")
```



### Address using Route

#### Pre-Req: Create Route

```shell
❯ oc create route passthrough grpc --service=grpc-service

❯ oc create route passthrough apicast-paintext --service=test-apicast-service --port=plaintext
❯ oc create route passthrough apicast-tls --service=test-apicast-service --port=tls
```

#### 

1. Client call to the Route of grpc service. Plaintext connection.

```shell
oc exec -ti client --  /go/bin/client --address $(oc get route -o jsonpath='{.spec.host}' grpc):443 --name "t1"
```

> <u>output</u>:
>
> ```shell
> 	- Address: 172.19.0.3:8043
> 	- Name: world
> 	- User_key: test
> 2023/01/17 14:09:26 Greeting: Hello t1
> ```

2. Client call to the Route of APIcast service.

   > Using port: **<u>plaintext</u>** 

```shell
oc exec -ti client --  /go/bin/client --address $(oc get route -o jsonpath='{.spec.host}' apicast-paintext):443 --name "t2"
```

> <u>output</u>:
>
> ```shell
> 	- Address: 172.19.0.3:8043
> 	- Name: world
> 	- User_key: test
> 2023/01/17 14:16:29 could not greet: rpc error: code = Unavailable desc = connection error: desc = "transport: authentication handshake failed: EOF"
> command terminated with exit code 1
> ```
>
> <u>output</u> of logs: `oc logs -f dc/apicast --tail=1`
>
> ```shell
> NONE!
> ```



2. > Using port: **<u>tls</u>** 

```shell
oc exec -ti client --  /go/bin/client --address $(oc get route -o jsonpath='{.spec.host}' apicast-tls):443 --name "t3"
```

> <u>output</u>:
>
> ```shell
> 	- Address: 172.19.0.3:8043
> 	- Name: world
> 	- User_key: test
> 2023/01/17 14:18:03 could not greet: rpc error: code = Unavailable desc = connection error: desc = "transport: authentication handshake failed: EOF"
> command terminated with exit code 1
> ```
>
> <u>output</u> of logs: `oc logs -f dc/apicast --tail=1`
>
> ```shell
> NONE!
> ```
>
> 
