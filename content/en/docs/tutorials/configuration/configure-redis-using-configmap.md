---
reviewers:
- eparis
- pmorie
title: Configure Redis using a ConfigMap
content_type: tutorial
weight: 30
---

<!-- overview -->

ConfigMaps are a Kubernetes mechanism that let you inject configuration data into application pods.
This guide demonstrates how to configure Redis using a ConfigMap and builds on the [Configure a Pod to use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/) task.

## What you'll learn

In this tutorial, you'll learn how to do the following tasks:

* Create a ConfigMap with Redis configuration values.
* Create a Redis Pod that mounts and uses the created ConfigMap.
* Verify that the configuration was correctly applied.

## Requirements

{{< include "task-tutorial-prereqs.md" >}}

* You are using `kubectl` 1.14 and above.
  <details>
   <summary>Check `kubectl` version</summary>
   {{< version-check >}}
  </details>
* You've [configured a Pod to use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/).

<!-- lessoncontent -->

## Step 1: Create and apply an empty ConfigMap

To create and apply a ConfigMap with an empty configuration block, you can use kubectl and an example Redis pod manifest.

1. Create a ConfigMap with an empty configuration block, using the following command:

    ```shell
    cat <<EOF >./example-redis-config.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: example-redis-config
    data:
      redis-config: ""
    EOF
    ```

1. Apply the empty ConfigMap, using the following command:

    ```shell
    kubectl apply -f example-redis-config.yaml
    ```

1. Apply an example Redis pod manifest, using the following command:

    ```shell
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
    ```

    {{< details summary="View the example Redis pod manifest" >}}

    {{% code_sample file="pods/config/redis-pod.yaml" %}}

    {{< /details >}}

   Examine the contents of the example Redis pod manifest and note the following:

    * A volume named `config` is created by `spec.volumes[1]`.
    * The `key` and `path` under `spec.volumes[1].configMap.items[0]` exposes the `redis-config` key from the
      `example-redis-config` ConfigMap as a file named `redis.conf` on the `config` volume.
    * The `config` volume is then mounted at `/redis-master` by `spec.containers[0].volumeMounts[1]`.

    The contents of the Redis pod have the net effect of exposing the data in `data.redis-config` from the `example-redis-config`
    ConfigMap as `/redis-master/redis.conf` inside the Pod.

1. Examine the created objects, using the following command:

    ```shell
    kubectl get pod/redis configmap/example-redis-config 
    ```

    You should see the following output:

    ```text
    NAME        READY   STATUS    RESTARTS   AGE
    pod/redis   1/1     Running   0          8s

    NAME                             DATA   AGE
    configmap/example-redis-config   1      14s
    ```

1. Check that the `redis-config` key in the `example-redis-config` ConfigMap is blank, using the following command:

    ```shell
    kubectl describe configmap/example-redis-config
    ```

    You should see an empty `redis-config` key:

    ```shell
    Name:         example-redis-config
    Namespace:    default
    Labels:       <none>
    Annotations:  <none>

    Data
    ====
    redis-config:
    ```

1. Use `kubectl exec` to enter the pod and run the `redis-cli` tool to check the current configuration, using the following command:

    ```shell
    kubectl exec -it redis -- redis-cli
    ```

1. Check `maxmemory`, using the following command:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory
    ```

    It should show the default value of 0:

    ```shell
    1) "maxmemory"
    2) "0"
    ```

1. Check `maxmemory-policy`, using the following command:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory-policy
    ```

    Which should also yield its default value of `noeviction`:

    ```shell
    1) "maxmemory-policy"
    2) "noeviction"
    ```

## Step 2: Add configuration values to the ConfigMap

To apply configuration values to the ConfigMap, you can use kubectl and an example ConfigMap.

1. Update the ConfigMap with the following content from an example ConfigMap:

    {{% code_sample file="pods/config/example-redis-config.yaml" %}}

1. Apply the updated ConfigMap, using the following command:

    ```shell
    kubectl apply -f example-redis-config.yaml
    ```

1. Confirm that the ConfigMap was updated, using the following command:

    ```shell
    kubectl describe configmap/example-redis-config
    ```

    You should see the configuration values we just added:

    ```shell
    Name:         example-redis-config
    Namespace:    default
    Labels:       <none>
    Annotations:  <none>

    Data
    ====
    redis-config:
    ----
    maxmemory 2mb
    maxmemory-policy allkeys-lru
    ```

1. Check the Redis Pod again using `redis-cli` via `kubectl exec` to see if the configuration was applied, using the following command:

    ```shell
    kubectl exec -it redis -- redis-cli
    ```

1. Check `maxmemory`, using the following command:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory
    ```

    It remains at the default value of 0:

    ```shell
    1) "maxmemory"
    2) "0"
    ```

1. Check that `maxmemory-policy` remains at the `noeviction` default setting, using the following command:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory-policy
    ```

    Returns:

    ```shell
    1) "maxmemory-policy"
    2) "noeviction"
    ```

The configuration values have not changed because the Pod needs to be restarted to grab updated
values from associated ConfigMaps.

## Step 3: Delete and recreate the Pod

To grab the updated values from the associated ConfigMaps, you can delete and recreate the Pod using kubectl.

1. Delete the Pod, using the following command:

    ```shell
    kubectl delete pod redis
    ```

1. Re-create the pod, using the following command:

    ```shell
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
    ```

1. Re-check the configuration values, using the following command:

    ```shell
    kubectl exec -it redis -- redis-cli
    ```

1. Check `maxmemory`, using the following command:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory
    ```

    It should now return the updated value of 2097152:

    ```shell
    1) "maxmemory"
    2) "2097152"
    ```

1. Check that `maxmemory-policy` has also been updated, using the following command:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory-policy
    ```

    It now reflects the desired value of `allkeys-lru`:

    ```shell
    3) "maxmemory-policy"
    4) "allkeys-lru"
    ```

## Step 4: Clean up your work

Delete the created resources, using the following command:

```shell
kubectl delete pod/redis configmap/example-redis-config
```

## Next steps

* Learn more about [ConfigMaps](/docs/tasks/configure-pod-container/configure-pod-configmap/).
* Follow an example of [Updating configuration via a ConfigMap](/docs/tutorials/configuration/updating-configuration-via-a-configmap/).
