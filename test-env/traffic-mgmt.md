Version 2 of the customers service has been developed, and it's time to deploy it to production.
Whereas version 1 returned a list of customer names, version 2 also includes each customer's city.

## Deploying customers, v2

We wish to deploy the new service but aren't yet ready to direct traffic to it.

It would be prudent to separate the task of deploying the new service from the task of directing traffic to it.

### Labels

The customers service is labeled with `app=customers`.

Verify this with:

```
k get pod -Lapp,version
```{{exec}}

Note the selector on the customers service:

```
k get svc customers -o wide
```{{exec}}

If we were to just deploy v2, the selector would match both versions.

### DestinationRules

We can inform Istio that two distinct subsets of the `customers` service exist, and we can use the `version` label as the discriminator.

```
cat customers-destinationrule.yaml
```{{exec}}

1. Apply the above destination rule to the cluster.

1. Verify that it's been applied.

    ```
    k get destinationrule
    ```{{exec}}

### VirtualServices

Armed with two distinct destinations, the `VirtualService` custom resource allows us to define a routing rule that sends all traffic to the v1 subset.

```
cat customers-virtualservice.yaml
```{{exec}}

Above, note how the route specifies subset v1.

1. Apply the virtual service to the cluster.

1. Verify that it's been applied.

    ```
    k get virtualservice 
    ```{{exec}}

### Finally deploy customers, v2

Apply the following Kubernetes deployment to the cluster.

```
cat customers-v2.yaml
```{{exec}}

### Check that traffic routes strictly to v1

1. Generate some traffic.

    ```
    INGRESS_CLUSTER_IP=$(kubectl get svc -n istio-system istio-ingressgateway -ojsonpath='{.spec.clusterIP}')
    ```{{exec}}

    ```
    siege --delay=3 --concurrent=3 --time=20M http://$INGRESS_CLUSTER_IP/
    ```{{exec}}

1. Open a separate terminal and expose the Kiali service

    ```
    k port-forward -n istio-system --address 0.0.0.0 service/kiali 20001:20001
    ```{{exec}}

    [Access Kiali]({{TRAFFIC_HOST1_20001}}/).

Take a look at the graph, and select the `default` namespace.

The graph should show all traffic going to v1.

## Route to customers, v2

We wish to proceed with caution.  Before customers can see version 2, we want to make sure that the service functions properly.

## Expose "debug" traffic to v2

Review this proposed updated routing specification.

```
cat customers-vs-debug.yaml
```{{exec}}

We are telling Istio to check an HTTP header:  if the `user-agent` is set to `debug`, route to v2, otherwise route to v1.

Open a new terminal and apply the above resource to the cluster; it will overwrite the currently defined virtualservice as both yamls use the same resource name.

```
k apply -f customers-vs-debug.yaml
```{{exec}}

### Test it

Expose the ingress gateway once more:

```
k port-forward -n istio-system --address 0.0.0.0 service/istio-ingressgateway 1234:80
```{{exec}}

[Access the gateway]({{TRAFFIC_HOST1_1234}}/).

We can tell v1 and v2 apart in that v2 displays not only customer names but also their city (in two columns).

If you're using Chrome or Firefox, you can customize the `user-agent` header as follows:

1. Open the browser's developer tools
2. Open the "three dots" menu, and select _More tools --> Network conditions_
3. The network conditions panel will open
4. Under _User agent_, uncheck _Use browser default_
5. Select _Custom..._ and in the text field enter `debug`

Refresh the page; traffic should be directed to v2.

#### Tip

If you refresh the page a good dozen times and then wait ~15-30 seconds, you should see some of that v2 traffic in Kiali.

## Canary

Well, v2 looks good; we decide to expose the new version to the public, but we're still prudent.

Start by siphoning 10% of traffic over to v2.

```
cat customers-vs-canary.yaml
```{{exec}}

Above, note the `weight` field specifying 10 percent of traffic to v2.
Kiali should now show traffic going to both v1 and v2.

- Apply the above resource.
- In your browser:  undo the user agent customization and refresh the page a bunch of times.

Most of the requests still go to v1, but some are directed to v2.

## Check Grafana

Before we open the floodgates, we wish to determine how v2 is fairing.

Expose Grafana:

```
k port-forward -n istio-system --address 0.0.0.0 service/grafana 3000:3000
```{{exec}}

[Access Grafana]({{TRAFFIC_HOST1_3000}}/).


In Grafana, visit the Istio Workload Dashboard and specifically look at the customers v2 workload.
Look at the request rate and the incoming success rate, also the latencies.

If all looks good, up the percentage from 90/10 to, say 50/50.

Watch the request volume change (you may need to click on the "refresh dashboard" button in the upper right-hand corner).

Finally, switch all traffic over to v2.

```
cat customers-virtualservice-final.yaml
```{{exec}}

After you apply the above yaml, go to your browser and make sure all requests land on v2 (2-column output).
Within a minute or so, the Kiali dashboard should also reflect the fact that all traffic is going to the customers v2 service.

Though it no longer receives any traffic, we decide to leave v1 running a while longer before retiring it.
