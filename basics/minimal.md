
The file `minimal-config.yaml` is just that: it contains the minimum configuration necessary to start envoy.

1. Review the file's contents:

    ```
    cat minimal-config.yaml
    ```{{exec}}

    envoy will listen on port 10000, and is configured with no filters.

1. Run envoy in the background:

    ```
    func-e run --config-path minimal-config.yaml &
    ```{{exec}}

1. Try to send a request to `localhost:10000`:

    ```
    curl -v localhost:10000
    ```{{exec}}

    envoy does not yet know how to route the request.

Stop envoy: bring the process back to the foreground with `fg`, then press `ctrl+c`.