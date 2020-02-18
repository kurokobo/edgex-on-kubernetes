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
1. Create some of Deployments
    ```bash
    $ kubectl create -f deployments/edgex-files-deployment.yaml
    $ kubectl wait --for=condition=available --timeout=60s deployment edgex-files

    $ kubectl create -f deployments/edgex-core-consul-deployment.yaml
    $ kubectl wait --for=condition=available --timeout=60s deployment edgex-core-consul
    ```
1. Initialize Key-Value store of Consul by running `edgex-core-config-seed`
    ```bash
    $ kubectl create -f pods/edgex-core-config-seed-pod.yaml
    $ kubectl wait --for=condition=initialized --timeout=60s pod edgex-core-config-seed
    ```
1. Change values to `0.0.0.0` for all of `/edgex/core/1.0/*/Service/Host` in your Consul service. TODO: find better way ...
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
    

## Preparing NFS Storage

Create sub directories under the root of exported directory.

```bash
$ mount -t nfs <your-nfs-server>:/path/to/export /path/to/mount
$ cd /path/to/mount
$ for i in $(seq -f %03g 100); do sudo mkdir ./pv$i; done
```
