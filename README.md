# Multi-LWS Gradual Rollout Demo

This demo show how 2 LeaderWorkerSet objects can gradually be updated
in-sync, one pod at a time in each LWS, by manually decrementing LWS
`partition` field value.

## Prerequisites

- kind
- [watch](https://man7.org/linux/man-pages/man1/watch.1.html)


## Steps

1. Create a kind cluster

    ```
    $ kind create cluster
    ```

1. Install the LWS operator v0.7.0

    ```
    $ kubectl apply --server-side -f https://github.com/kubernetes-sigs/lws/releases/download/0.7.0/manifests.yaml
    ```

1. Wait for the LWS operator to be ready

    ```
    $ kubectl wait deploy/lws-controller-manager -n lws-system --for=condition=available --timeout=5m

    deployment.apps/lws-controller-manager condition met
    ```

1. Deploy 2 LWS in the default namespace

    ```
    $ kubectl apply -f config/1.26
    ```

    Both LWSs are configured to run 3 nginx **v1.26** replicas.

1. Open a new terminal and watch pods until their status is RUNNING.

    ```
    $ watch -n 1 kubectl get pods

    NAME                        READY   STATUS    RESTARTS   AGE
    leaderworkerset-decode-0    1/1     Running   0          22s
    leaderworkerset-decode-1    1/1     Running   0          22s
    leaderworkerset-decode-2    1/1     Running   0          22s
    leaderworkerset-prefill-0   1/1     Running   0          22s
    leaderworkerset-prefill-1   1/1     Running   0          22s
    leaderworkerset-prefill-2   1/1     Running   0          22s
    ```
1. In the previous terminal, update both LWS to use nginx **1.27**


    ```
    $ kubectl apply -f config/1.27
    ```

    In the watch terminal, notice no pods are being terminated and created:
    - `maxSurge` set to 0  prevents the number of replicas to go above the number of replicas (3 in this example)
    - `partition` set to 3 prevents replicas of ordinal index below 3 to be updated

1. Allow replica rolling update of ordinal index 2 for both LWS (decode and prefill)

    ```
    $ kubectl patch lws leaderworkerset-decode --type='json' -p='[{"op": "replace", "path": "/spec/rolloutStrategy/rollingUpdateConfiguration/partition", "value": 2}]' & kubectl patch lws leaderworkerset-prefill --type='json' -p='[{"op": "replace", "path": "/spec/rolloutStrategy/rollingUpdateConfiguration/partition", "value": 2}]'
    ```

    Notice is the watch terminal, both leaderworkerset-decode-2 and leaderworkerset-prefill-2 are being updated.

1. Allow replica rolling update of ordinal index 1 for both LWS (decode and prefill)

    ```
    $ kubectl patch lws leaderworkerset-decode --type='json' -p='[{"op": "replace", "path": "/spec/rolloutStrategy/rollingUpdateConfiguration/partition", "value": 1}]' & kubectl patch lws leaderworkerset-prefill --type='json' -p='[{"op": "replace", "path": "/spec/rolloutStrategy/rollingUpdateConfiguration/partition", "value": 1}]'
    ```

    Repeat until all replicas have been updated.
