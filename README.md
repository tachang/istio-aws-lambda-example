# Calling AWS Lambda Functions from Istio

Example on how to get Istio to call AWS Lambda functions.

## Running

To build all the resources:

`kustomize build .`

To apply them to your cluster:

`kubectl apply -k .`

## Testing framework

I used k3d for testing this example. Once `k3d` is up and running I made sure the load balancer was accessible. Int his case I forwarded port 9080 into the k3d cluster I created.

To figure out what port was forwarded I did a `docker ps` to identity the pod:

```
"/bin/sh -c nginx-pr…"   8 days ago    Up 8 days
0.0.0.0:9080->80/tcp, :::9080->80/tcp, 0.0.0.0:9443->443/tcp, :::9443->443/tcp, 0.0.0.0:44725->6443/tcp       k3d-k3s-default-serverlb
```

This allowed access to http://<server_name>:9080/.

For testing the lambdas I would use curl:

```
curl http://<server_name>:9080/cat
curl http://<server_name>:9080/dog
```

## Debugging

-   When debugging the configuration make sure to run curl multiple times over and over after applying. I noticed that some requests would work and then some would fail then eventually stabilize after subsequent requests. This is probably because the EnvoyFilter did not fully sync up yet even though `istioctl proxy-config` did show it as synced. Running `istioctl proxy-config route` though allowed me to see if the route was there or not.

-   `istioctl proxy-status` should have all the resources as "SYNCED". If any are showing an error that is the first place to look at.

-   There is some weirdness is how the aws_lambda filter calculates the signature. It seems to take into account the port when calculating the signature. Hard coding the request Host header to example.com seems to work as well as stripping the port before the calculation via `strip_any_host_port: true`.

-   When updating the IAM keys I found I had to delete the istio-ingressgateway and istiod pods in the istio-system namespace before the AWS lambda filter started to pick up the new keys.

#### Check to see if the listener configuration on the envoy proxy is correct.

The output of `istioctl proxy-config listener httpbin-85d76b4bb6-4v65k.default -o yaml` will dump the listener configuration for a specify envoy pod. You should see the word `lambda` somewhere as well as your specific route configuration.

#### Get the istio-proxy logs on the pod and tail them. Make sure you are at the debug tracing level.

`kubectl logs -f -c istio-proxy --since=1s httpbin-85d76b4bb6-4v65k`

An example of what you'll see if you can't retrieve AWS credentials from metadata:

```
$ kubectl logs -f -c istio-proxy --since=1s httpbin-85d76b4bb6-4v65k
2022-08-01T21:42:51.857821Z     error   envoy aws       Could not retrieve credentials listing from the instance metadata
```

#### How do I change the log level for the istio-proxy within a deployment?

Change the istio-proxy log level in the deployment by adding the annotation

"sidecar.istio.io/logLevel": debug

$ kubectl edit deployment $<YOUR_DEPLOYMENT>

```
template:
metadata:
  annotations:
    "sidecar.istio.io/logLevel": debug # level can be: trace, debug, info, warning, error, critical, off
```

If this doesn't take don't forget to delete the pods so they get recreated from the deployment.

#### What should I see in the istio-proxy sidecar logs?

You are looking for debug level logs that say "envoy http". Here is an example:

```
2022-08-01T21:52:11.211705Z debug envoy http [C22][s14188986547451111202] request headers complete (end_stream=true):
':authority', 'example.com:9080'
':path', '/cat'
':method', 'GET'
'user-agent', 'curl/7.69.1'
'accept', '_/_'
'x-forwarded-for', '10.42.5.1'
'x-forwarded-proto', 'http'
'x-envoy-internal', 'true'
'x-envoy-decorator-operation', 'httpbin.default.svc.cluster.local:8000/\*'
'x-envoy-peer-metadata', 'ChQKDkFQUF9DT05UQUlORVJTEgIaAAoaCgpDTFVTVEVSX0lEEgwaCkt1YmVybmV0ZXMKGQoNSVNUSU9fVkVSU0lPThIIGgYxLjExLjAKvgMKBkxBQkVMUxKzAyqwAwodCgNhcHASFhoUaXN0aW8taW5ncmVzc2dhdGV3YXkKEwo
FY2hhcnQSChoIZ2F0ZXdheXMKFAoIaGVyaXRhZ2USCBoGVGlsbGVyCjYKKWluc3RhbGwub3BlcmF0b3IuaXN0aW8uaW8vb3duaW5nLXJlc291cmNlEgkaB3Vua25vd24KGQoFaXN0aW8SEBoOaW5ncmVzc2dhdGV3YXkKGQoMaXN0aW8uaW8vcmV2EgkaB2RlZmF1
bHQKMAobb3BlcmF0b3IuaXN0aW8uaW8vY29tcG9uZW50EhEaD0luZ3Jlc3NHYXRld2F5cwogChFwb2QtdGVtcGxhdGUtaGFzaBILGglkY2RjNThkNjgKEgoHcmVsZWFzZRIHGgVpc3Rpbwo5Ch9zZXJ2aWNlLmlzdGlvLmlvL2Nhbm9uaWNhbC1uYW1lEhYaFGlzd
GlvLWluZ3Jlc3NnYXRld2F5Ci8KI3NlcnZpY2UuaXN0aW8uaW8vY2Fub25pY2FsLXJldmlzaW9uEggaBmxhdGVzdAoiChdzaWRlY2FyLmlzdGlvLmlvL2luamVjdBIHGgVmYWxzZQoaCgdNRVNIX0lEEg8aDWNsdXN0ZXIubG9jYWwKLgoETkFNRRImGiRpc3Rpby
1pbmdyZXNzZ2F0ZXdheS1kY2RjNThkNjgtOWxrcGMKGwoJTkFNRVNQQUNFEg4aDGlzdGlvLXN5c3RlbQpdCgVPV05FUhJUGlJrdWJlcm5ldGVzOi8vYXBpcy9hcHBzL3YxL25hbWVzcGFjZXMvaXN0aW8tc3lzdGVtL2RlcGxveW1lbnRzL2lzdGlvLWluZ3Jlc3N
nYXRld2F5ChcKEVBMQVRGT1JNX01FVEFEQVRBEgIqAAonCg1XT1JLTE9BRF9OQU1FEhYaFGlzdGlvLWluZ3Jlc3NnYXRld2F5'
'x-envoy-peer-metadata-id', 'router~10.42.5.37~istio-ingressgateway-dcdc58d68-9lkpc.istio-system~istio-system.svc.cluster.local'
'x-b3-traceid', 'd5db84e2ee3352c5a2c2b701349ccd4b'
'x-b3-spanid', 'a2c2b701349ccd4b'
'x-b3-sampled', '0'

2022-08-01T21:52:11.211740Z debug envoy http [C22][s14188986547451111202] request end stream
2022-08-01T21:52:11.213510Z debug envoy aws Getting AWS credentials from the environment
2022-08-01T21:52:11.213555Z debug envoy aws Getting AWS credentials from the instance metadata
```

#### The request signature we calculated does not match the signature you provided.

The full message you get back from AWS:

```
{"message":"The request signature we calculated does not match the signature you provided. Check your AWS Secret Access Key and signing method. Consult the service documentation for details."}
```

This is a mismatch of what is being hashed over vs what AWS is hashing. The culprit is usually extra headers or port numbers.

For this example I was using k3d. With k3d I set the port 9080 to forward into the load balancer on port 80. So while I access my cluster via port 9080 the load balancer for k3d thinks all the requests are originating via port 80.

When testing I passed in the host header I would have used if k3d was not in the mix:

```
curl -H "Host: example.com" http://example.com:9080/cat

```

You need to provide a host header, otherwise the signature calculation takes the host with the port into account.

> Usually that's an issue with curl on the command line but not when called from a service or when you specify the port individually, for example via libcurl API.

2022-08-01T22:27:36.861326Z debug envoy router [C16][s13163796387753860385] router decoding headers:  
':authority', 'r1.zspin.com:9080'  
':path', '/2015-03-31/functions/hello-world-cat/invocations'  
':method', 'POST'  
':scheme', 'https'  
'user-agent', 'curl/7.69.1'  
'accept', '_/_'  
'x-forwarded-proto', 'https'  
'x-request-id', '2f29c764-5cf8-4e3d-8d4c-cb32a9444386'  
'x-forwarded-client-cert', 'By=spiffe://cluster.local/ns/default/sa/httpbin;Hash=4d6f495c07d35b84987331dd0432ccad666f94d5a59d0d9cfb328d7f3dfbd9b2;Subject="";URI=spiffe://cluster.local/ns/istio-syst
em/sa/istio-ingressgateway-service-account'  
'x-amz-invocation-type', 'RequestResponse'  
'content-length', '477'  
'content-type', 'application/json'  
'x-amz-content-sha256', '7aefa745ba2b6b2e0c1294190bc3379ecebec116539ce9e11ed9c1bd4c081798'  
'x-amz-date', '20220801T222700Z'  
'authorization', 'AWS4-HMAC-SHA256 Credential=AKIAIB7J2AOEOY2YM7OA/20220801/us-east-1/lambda/aws4_request, SignedHeaders=accept;content-length;content-type;host;user-agent;x-amz-content-sha256;x-am
z-date;x-amz-invocation-type;x-forwarded-client-cert;x-request-id, Signature=bc2d30f0e6252cdf83068b28129aba21e9430264b3a3119397adbbcd2d3c35a3'  
'x-envoy-expected-rq-timeout-ms', '15000'  
'x-b3-traceid', '2806911aa337c0e2ce32312eaca11aa3'  
'x-b3-spanid', 'ce32312eaca11aa3'  
'x-b3-sampled', '0'

2022-08-01T22:27:36.860686Z debug envoy aws Getting AWS credentials from the environment  
2022-08-01T22:27:36.860742Z debug envoy aws Found following AWS credentials in the environment: AWS_ACCESS_KEY_ID=AKIAIB7J2AOEOY2YM7OA, AWS_SECRET_ACCESS_KEY=**\***, AWS_SESSION_TOKEN=  
2022-08-01T22:27:36.860847Z debug envoy http Canonical request:  
POST  
/2015-03-31/functions/hello-world-cat/invocations

accept:_/_  
content-length:477  
content-type:application/json  
host:r1.zspin.com:9080  
user-agent:curl/7.69.1  
x-amz-content-sha256:7aefa745ba2b6b2e0c1294190bc3379ecebec116539ce9e11ed9c1bd4c081798  
x-amz-date:20220801T222700Z  
x-amz-invocation-type:RequestResponse  
x-forwarded-client-cert:By=spiffe://cluster.local/ns/default/sa/httpbin;Hash=4d6f495c07d35b84987331dd0432ccad666f94d5a59d0d9cfb328d7f3dfbd9b2;Subject="";URI=spiffe://cluster.local/ns/istio-system/s
a/istio-ingressgateway-service-account  
x-request-id:2f29c764-5cf8-4e3d-8d4c-cb32a9444386

accept;content-length;content-type;host;user-agent;x-amz-content-sha256;x-amz-date;x-amz-invocation-type;x-forwarded-client-cert;x-request-id  
7aefa745ba2b6b2e0c1294190bc3379ecebec116539ce9e11ed9c1bd4c081798  
2022-08-01T22:27:36.860933Z debug envoy http String to sign:  
AWS4-HMAC-SHA256  
20220801T222700Z  
20220801/us-east-1/lambda/aws4_request  
3fbc0a8a24de6e61b6de6668d47fc2c00521009cce34528e5e2e54b5827dff49  
2022-08-01T22:27:36.860997Z debug envoy http Signing request with: AWS4-HMAC-SHA256 Credential=AKIAIB7J2AOEOY2YM7OA/20220801/us-east-1/lambda/aws4_request, SignedHeaders=accept;content-l
ength;content-type;host;user-agent;x-amz-content-sha256;x-amz-date;x-amz-invocation-type;x-forwarded-client-cert;x-request-id, Signature=bc2d30f0e6252cdf83068b28129aba21e9430264b3a3119397adbbcd2d3c
35a3

#### What does a working AWS lambda call in the istio-proxy log look like

2022-08-01T22:51:46.460864Z debug envoy http String to sign:  
AWS4-HMAC-SHA256  
20220801T225100Z  
20220801/us-east-1/lambda/aws4*request  
6cb3442f8e8f88c6b2a89efe4efcb0ca416c48e44ef38ae376ef2a83a022d194  
2022-08-01T22:51:46.460934Z debug envoy http Signing request with: AWS4-HMAC-SHA256 Credential=AKIAIB7J2AOEOY2YM7OA/20220801/us-east-1/lambda/aws4_request, SignedHeaders=accept;content-l
ength;content-type;host;user-agent;x-amz-content-sha256;x-amz-date;x-amz-invocation-type;x-forwarded-client-cert;x-request-id, Signature=16b63ff0a4f20a893bd9ae2539aa400d45f0203ee14ed66e54f28e4e95e9
52db  
2022-08-01T22:51:46.460976Z debug envoy router [C453][s18103120217120842764] cluster 'lambda_egress_gateway' match for URL '/2015-03-31/functions/hello-world-cat/invocations'  
2022-08-01T22:51:46.461099Z debug envoy router [C453][s18103120217120842764] router decoding headers:  
':authority', 'r1.zspin.com'  
':path', '/2015-03-31/functions/hello-world-cat/invocations'  
':method', 'POST'  
':scheme', 'https'  
'user-agent', 'curl/7.69.1'  
'accept', '*/\_'  
'x-request-id', '3ba4bc94-3b9b-4758-b883-a08befa9e1fe'  
'x-forwarded-proto', 'https'  
'x-forwarded-client-cert', 'By=spiffe://cluster.local/ns/default/sa/httpbin;Hash=4d6f495c07d35b84987331dd0432ccad666f94d5a59d0d9cfb328d7f3dfbd9b2;Subject="";URI=spiffe://cluster.local/ns/istio-syst
em/sa/istio-ingressgateway-service-account'  
'x-amz-invocation-type', 'RequestResponse'  
'content-length', '477'  
'content-type', 'application/json'  
'x-amz-content-sha256', '3358c2a4872f610ecce754dc4fa5e096d370743baaa08eecffaa56fe22dad64e'  
'x-amz-date', '20220801T225100Z'  
'authorization', 'AWS4-HMAC-SHA256 Credential=AKIAIB7J2AOEOY2YM7OA/20220801/us-east-1/lambda/aws4_request, SignedHeaders=accept;content-length;content-type;host;user-agent;x-amz-content-sha256;x-am
z-date;x-amz-invocation-type;x-forwarded-client-cert;x-request-id, Signature=16b63ff0a4f20a893bd9ae2539aa400d45f0203ee14ed66e54f28e4e95e952db'  
'x-envoy-expected-rq-timeout-ms', '15000'  
'x-b3-traceid', '29663dfe849484bc13fa697c201ebeec'  
'x-b3-spanid', '13fa697c201ebeec'  
'x-b3-sampled', '0'
