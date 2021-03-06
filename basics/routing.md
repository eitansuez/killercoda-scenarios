In the previous step, we saw envoy load balance between two endpoints belonging to a single cluster.

In this step, we configure those two backing services differently:  we model two distinct clusters each with one endpoint.

1. Review the contents of the envoy configuration file named `routing.yaml`:

    ```
    cat routing.yaml
    ```{{exec}}

    The route configuration now uses a path prefix to determine which backend cluster to target.

    To help distinguish between the two backends, requests to the `/one` prefix will be routed to the `/ip` endpoint of the first cluster (`httpbin-1`), while rquests to the `/two` prefix go to the `/user-agent` endpoint of the second cluster.

1. Run envoy in the background:

    ```
    func-e run --config-path routing.yaml &
    ```{{exec}}

1. Check that the two docker containers are still running:

    ```
    docker ps
    ```{{exec}}

1. Send a request to envoy, targeting the first backend:

    ```
    curl localhost:10000/one
    ```{{exec}}

    And another, targeting the second backend:

    ```
    curl localhost:10000/two
    ```{{exec}}

Stop envoy: bring the process back to the foreground with `fg`, then press `ctrl+c`.