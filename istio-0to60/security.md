In this lab we explore some of the security features of the Istio service mesh.

## Mutual TLS

By default, Istio is configured such that when a service is deployed onto the mesh, it will take advantage of mutual TLS:

- the service is given an identity as a function of its associated service account and namespace
- an x.509 certificate is issued to the workload (and regularly rotated) and used to identify the workload in calls to other services

In the observability lab, we looked at the Kiali dashboard and noted the lock icons indicating that traffic was secured with mTLS.

### Can a workload receive plain-text requests?

We can test whether a mesh workload, such as the customers service, will allow a plain-text request as follows:

1. Create a separate namespace that is not configured with automatic injection.

    ```
    k create ns other-ns
    ```{{exec}}

1. Deploy `sleep` to that namespace

    ```
    k apply -f sleep.yaml -n other-ns
    ```{{exec}}

1. Verify that the sleep pod has no sidecars:

    ```
    k get pod -n other-ns
    ```{{exec}}

1. Call the customer service from that pod:

    ```
    kubectl exec -n other-ns deploy/sleep -- curl -s customers.default | jq
    ```{{exec}}

The output should look like a list of customers in JSON format.

We conclude that Istio is configured by default to allow plain-text request.
This is called _permissive mode_ and is specifically designed to allow services that have not yet fully on-boarded onto the mesh to participate.

### Enable strict mode

Istio provides the `PeerAuthentication` custom resource to define peer authentication policy.

1. Review the following peer authentication policy:

    ```
    cat mtls-strict.yaml
    ```{{exec}}

    Strict mTls can be enabled globally by setting the namespace to the name of the Istio root namespace, which by default is `istio-system`

1. Apply the policy:

    ```
    k apply -f mtls-strict.yaml
    ```{{exec}}

1. Verify that the peer authentication has been applied.

    ```
    k get peerauthentication
    ```{{exec}}

### Verify that plain-text requests are no longer permitted

```
k exec -n other-ns deploy/sleep -- curl customers.default
```{{exec}}

The console output should indicate that the _connection was reset by peer_.


### Inspecting a workload certificate

Capture the certificate returned by the `customers` workload:

```
kubectl exec deploy/sleep -c istio-proxy -- openssl s_client -showcerts -connect customers:80 > cert.txt
```{{exec}}

Inspect the certificate:

```
openssl x509 -in cert.txt -text -noout
```{{exec}}

The certificate's _Subject Alternative Name_ field contains the spiffe URI.


## Security in depth

Another important layer of security is to define an authorization policy, in which we allow only specific services to communicate with other services.

At the moment, any container can, for example, call the customers service or the web-frontend service.

1. Call the `customers` service.

    ```
    k exec deploy/sleep -- curl -s customers
    ```{{exec}}

1. Call the `web-frontend` service.

    ```
    k exec deploy/sleep -- curl -s web-frontend | head
    ```{{exec}}

Both calls succeed.

We wish to apply a policy in which _only `web-frontend` is allowed to call `customers`_, and _only the ingress gateway can call `web-frontend`_.

Study the below authorization policy.

```
cat authz-policy-customers.yaml
```{{exec}}

- The `selector` section specifies that the policy applies to the `customers` service.
- Note how the rules have a "from: source: " section indicating who is allowed in.
- The nomenclature for the value of the `principals` field comes from the [spiffe](https://spiffe.io/docs/latest/spiffe-about/overview/) standard.  Note how it captures the service account name and namespace associated with the `web-frontend` service.  This identify is associated with the x.509 certificate used by each service when making secure mTls calls to one another.

Tasks:

- [ ] Apply the policy to your cluster.
- [ ] Verify that you are no longer able to reach the `customers` pod from the `sleep` pod


### Challenge

Can you come up with a similar authorization policy for `web-frontend`?

- Use a copy of the `customers` authorization policy as a starting point
- Give the resource an apt name
- Revise the selector to match the `web-frontend` service
- Revise the rule to match the principal of the ingress gateway

#### Hint

The ingress gateway has its own identity.

Here is a command which can help you find the name of the service account associated with its identity:

```
k get pod -n istio-system -l app=istio-ingressgateway -o yaml | grep serviceAccountName
```{{exec}}

Use this service account name together with the namespace that the ingress gateway is running in, to specify the value for the `principals` field.


### Test it

Don't forget to verify that the policy is enforced.

- Call both services again from the sleep pod and ensure communication is no longer allowed.
- The console output should contain the message _RBAC: access denied_.

Conversely, verify that the Ingress Gateway can still call the `web-frontend` service:

```
INGRESS_CLUSTER_IP=$(kubectl get svc -n istio-system istio-ingressgateway -ojsonpath='{.spec.clusterIP}')
```{{exec}}

Then:

```
k exec deploy/sleep -- curl -s -I ${INGRESS_CLUSTER_IP}
```{{exec}}


## Next

In the next lab we show how to use Istio's traffic management features to upgrade the customers service with zero downtime.
