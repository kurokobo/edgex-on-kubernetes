# EdgeX Foundry on Kubernetes

Kubernetes manifests for EdgeX Foundry Fuji release based on [`docker-compose-fuji-no-secty.yml`](https://github.com/edgexfoundry/developer-scripts/blob/master/releases/fuji/compose-files/docker-compose-fuji-no-secty.yml).

Please note these manifests are working in progress.

## Quick instruction

1. Create Persistent Volumes
    1. Prepare your PVs. This repository includes only manifests for PVs for types of nfs storage.
    1. Modify manifests of Persistent Volumes in `persistentvolumes` directory to suit your environment.
    1. Create PVs.
        ```bash
        $ ls persistentvolumes/nfs/*.yaml | xargs -n 1 kubectl create -f
        $ kubectl get pv
        ```
1. Create Persistent Volume Claims
    ```bash
    $ ls persistentvolumeclaims/*.yaml | xargs -n 1 kubectl create -f
    $ kubectl get pv,pvc
    ```
1. Create Services
    ```bash
    $ ls services/*.yaml | xargs -n 1 kubectl create -f
    $ kubectl get svc
    ```
1. Initialize shared volumes by running `edgex-files`
    ```bash
    $ kubectl create -f pods/edgex-files-pod.yaml
    $ while [[ $(kubectl get pods edgex-files -o "jsonpath={.status.phase}") != "Succeeded" ]]; do sleep 1; done
    ```
1. Create Consul
    ```bash
    $ kubectl create -f deployments/edgex-core-consul-deployment.yaml
    $ kubectl wait --for=condition=available --timeout=60s deployment edgex-core-consul
    ```
1. Initialize Key-Value store of Consul by running `edgex-core-config-seed`
    ```bash
    $ kubectl create -f pods/edgex-core-config-seed-pod.yaml
    $ while [[ $(kubectl get pods edgex-core-config-seed -o "jsonpath={.status.phase}") != "Succeeded" ]]; do sleep 1; done
    ```
1. Create rest of Deployments
    ```bash
    $ kubectl create -f deployments/edgex-mongo-deployment.yaml
    $ kubectl wait --for=condition=available --timeout=60s deployment edgex-mongo

    $ kubectl create -f deployments/edgex-support-logging-deployment.yaml
    $ kubectl wait --for=condition=available --timeout=60s deployment edgex-support-logging

    $ kubectl create -f deployments/edgex-support-notifications-deployment.yaml
    $ kubectl wait --for=condition=available --timeout=60s deployment edgex-support-notifications

    $ kubectl create -f deployments/edgex-core-metadata-deployment.yaml
    $ kubectl wait --for=condition=available --timeout=60s deployment edgex-core-metadata

    $ kubectl create -f deployments/edgex-core-data-deployment.yaml
    $ kubectl wait --for=condition=available --timeout=60s deployment edgex-core-data

    $ kubectl create -f deployments/edgex-core-command-deployment.yaml
    $ kubectl wait --for=condition=available --timeout=60s deployment edgex-core-command

    $ kubectl create -f deployments/edgex-support-scheduler-deployment.yaml
    $ kubectl wait --for=condition=available --timeout=60s deployment edgex-support-scheduler

    $ kubectl create -f deployments/edgex-support-rulesengine-deployment.yaml
    $ kubectl wait --for=condition=available --timeout=60s deployment edgex-support-rulesengine

    $ kubectl create -f deployments/edgex-app-service-configurable-rules-deployment.yaml
    $ kubectl wait --for=condition=available --timeout=60s deployment edgex-app-service-configurable-rules

    $ kubectl create -f deployments/edgex-device-virtual-deployment.yaml
    $ kubectl wait --for=condition=available --timeout=60s deployment edgex-device-virtual

    $ kubectl create -f deployments/edgex-sys-mgmt-agent-deployment.yaml
    $ kubectl wait --for=condition=available --timeout=60s deployment edgex-sys-mgmt-agent

    $ kubectl create -f deployments/edgex-ui-go-deployment.yaml
    $ kubectl wait --for=condition=available --timeout=60s deployment edgex-ui-go
    ```
    

## Tips


### Preparing NFS Storage

Create sub directories under the root of exported directory.

```bash
$ mount -t nfs <your-nfs-server>:/path/to/export /path/to/mount
$ cd /path/to/mount
$ for i in $(seq -f %03g 100); do sudo mkdir ./pv$i; done
```


### Keeping the web server alive in the containers

Without special consideration, otherwise, the web servers in the containers stop immediately after its start.

Because of this, all API calls from other microservices and all health checks from consul will fail.

```bash
$ kubectl logs edgex-support-logging-68677766-5wj85
...
ts=2020-02-24T06:48:44.432322938Z app=edgex-support-logging source=message.go:58 msg="Service started in: 40.075685ms"
level=INFO ts=2020-02-24T06:48:44.432779288Z app=edgex-support-logging source=httpserver.go:80 msg="Web server stopped"
```

This is an example of `edgex-support-logging` but this issue will occur any of microservices in the `edgex-go` repository like `edgex-core-command`, `edgex-core-date`, and so on.

This is a problem caused by the following reasons.

* The web server tries to start a listener on `<hostname>:<port>`.
* This `<hostname>` has to be resolved to an IP address.  In this case, the `<hostname>` must be resolved to the IP address of the Pod.
* By default, `/etc/hosts` file in the container contains a correct pair of IP address and hostname of the Pod. This is the designed behavior of the Kubernetes.
* However, there is no `/etc/nsswitch.conf` in the container because the base image of this container is `scratch`.
* The default resolvers of Golang ignore `/etc/hosts` file if `/etc/nsswitch.conf` does not exist.
* Because of this, the `<hostname>` are resolved to one of `ClusterIP` incorrectly according to the Kubernetes name resolution mechanism.
* The `ClusterIP` is not visible to processes in the container. The process can't start a listener on the specified `<hostname>` and then stopped.

There are several workarounds to avoid this issue.

1. Specify `GODEBUG` environment variable with the value `netdns=cgo`. This will change the resolver of Golang to the CGO resolver which referes `/etc/hosts` instead of the pure Go resolver.
1. Place `/etc/nsswitch.conf` which contains `hosts: files dns`.
1. Modify the values to `0.0.0.0` for all of `/edgex/core/1.0/*/Service/Host` in your Consul service before we start other microservices.

The manifests in this repository adopt the first workaround.