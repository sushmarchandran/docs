---
template: overrides/main.html
---

# Conformance Tutorial

!!! tip ""
    Perform an Iter8-Knative experiment with [`Conformance`](/concepts/experimentationstrategies/#testing-pattern) testing.
    
    ![Canary](/assets/images/conformance.png)

You will create the following resources in this tutorial.

1. A **Knative app (service)** with a single version (revision).
2. A **fortio-based traffic generator** that simulates user requests.
3. An **Iter8 experiment** that verifies that `baseline` satisfies mean latency, 95th percentile tail latency, and error rate `objectives`.

??? warning "Before you begin, you will need ... "
    **Kubernetes cluster:** Ensure that you have Kubernetes cluster with Iter8 and Knative installed. You can do this by following Steps 1, 2, and 3 of [the quick start tutorial for Knative](/getting-started/quick-start/with-knative/).

    **Cleanup:** If you ran an Iter8 tutorial earlier, run the associated cleanup step.

    **ITER8:** Ensure that `ITER8` environment variable is set to the root directory of your cloned Iter8 repo. See [Step 2 of the quick start tutorial for Knative](/getting-started/quick-start/with-knative/#2-clone-repo) for example.

    **[iter8ctl](/getting-started/install/#step-4-install-iter8ctl):** This tutorial uses iter8ctl.

## 1. Create app
```shell
kubectl apply -f $ITER8/samples/knative/conformance/baseline.yaml
```

??? info "Look inside baseline.yaml"
    ```yaml linenums="1"
    apiVersion: serving.knative.dev/v1
    kind: Service
    metadata:
    name: sample-app 
    namespace: default 
    spec:
    template:
        metadata:
        name: sample-app-v1
        spec:
        containers:
        - image: gcr.io/knative-samples/knative-route-demo:blue 
            env:
            - name: T_VERSION
            value: "blue"
    ```

## 2. Generate requests
```shell
kubectl wait --for=condition=Ready ksvc/sample-app
URL_VALUE=$(kubectl get ksvc sample-app -o json | jq .status.address.url)
sed "s+URL_VALUE+${URL_VALUE}+g" $ITER8/samples/knative/conformance/fortio.yaml | kubectl apply -f -
```

## 3. Create Iter8 experiment
```shell
kubectl apply -f $ITER8/samples/knative/conformance/experiment.yaml
```
??? info "Look inside experiment.yaml"
    ```yaml
    apiVersion: iter8.tools/v2alpha1
    kind: Experiment
    metadata:
      name: conformance-sample
    spec:
      # target identifies the knative service under experimentation using its fully qualified name
      target: default/sample-app
      strategy:
        # this experiment will perform a conformance test
        testingPattern: Conformance
        actions:
          start: # run the following sequence of tasks at the start of the experiment
          - library: knative
            task: init-experiment
      criteria:
        # mean latency of version should be under 50 milliseconds
        # 95th percentile latency should be under 100 milliseconds
        # error rate should be under 1%
        objectives: 
        - metric: mean-latency
          upperLimit: 50
        - metric: 95th-percentile-tail-latency
          upperLimit: 100
        - metric: error-rate
          upperLimit: "0.01"
      duration:
        intervalSeconds: 10
        iterationsPerLoop: 10
      versionInfo:
        # information about app versions used in this experiment
        baseline:
          name: current
          variables:
          - name: revision
            value: sample-app-v1  
    ```

## 4. Observe experiment

Observe the experiment in realtime. Paste commands from the tabs below in separate terminals.

=== "iter8ctl"
    Periodically describe the experiment.
        ```shell
        while clear; do
        kubectl get experiment conformance-sample -o yaml | iter8ctl describe -f -
        sleep 4
        done
        ```

    ??? info "iter8ctl output"
        iter8ctl output will be similar to the following.
        ```shell
        ****** Overview ******
        Experiment name: conformance-sample
        Experiment namespace: default
        Target: default/sample-app
        Testing pattern: Conformance
        Deployment pattern: Progressive

        ****** Progress Summary ******
        Experiment stage: Running
        Number of completed iterations: 3

        ****** Winner Assessment ******
        Winning version: not found
        Recommended baseline: current

        ****** Objective Assessment ******
        +--------------------------------+---------+
        |           OBJECTIVE            | CURRENT |
        +--------------------------------+---------+
        | mean-latency <= 50.000         | true    |
        +--------------------------------+---------+
        | 95th-percentile-tail-latency   | true    |
        | <= 100.000                     |         |
        +--------------------------------+---------+
        | error-rate <= 0.010            | true    |
        +--------------------------------+---------+

        ****** Metrics Assessment ******
        +--------------------------------+---------+
        |             METRIC             | CURRENT |
        +--------------------------------+---------+
        | request-count                  | 448.000 |
        +--------------------------------+---------+
        | mean-latency (milliseconds)    |   1.338 |
        +--------------------------------+---------+
        | 95th-percentile-tail-latency   |   4.770 |
        | (milliseconds)                 |         |
        +--------------------------------+---------+
        | error-rate                     |   0.000 |
        +--------------------------------+---------+
        ```
        When the experiment completes (in ~ 2 mins), you will see the experiment stage change from `Running` to `Completed`.

=== "kubectl get experiment"

    ```shell
    kubectl get experiment conformance-sample --watch
    ```

    ??? info "kubectl get experiment output"
        kubectl output will be similar to the following.
        ```shell
        NAME                 TYPE          TARGET               STAGE          COMPLETED ITERATIONS   MESSAGE    
        conformance-sample   Conformance   default/sample-app   Running        0                      StartHandlerLaunched: Start handler 'start' launched
        conformance-sample   Conformance   default/sample-app   Running        1                      IterationUpdate: Completed Iteration 1
        conformance-sample   Conformance   default/sample-app   Running        2                      IterationUpdate: Completed Iteration 2
        conformance-sample   Conformance   default/sample-app   Running        3                      IterationUpdate: Completed Iteration 3
        conformance-sample   Conformance   default/sample-app   Running        4                      IterationUpdate: Completed Iteration 4
        conformance-sample   Conformance   default/sample-app   Running        5                      IterationUpdate: Completed Iteration 5
        conformance-sample   Conformance   default/sample-app   Running        6                      IterationUpdate: Completed Iteration 6
        conformance-sample   Conformance   default/sample-app   Running        7                      IterationUpdate: Completed Iteration 7
        ```
        When the experiment completes (in ~ 2 mins), you will see the experiment stage change from `Running` to `Completed`.

## 5. Cleanup

```shell
kubectl delete -f $ITER8/samples/knative/conformance/fortio.yaml
kubectl delete -f $ITER8/samples/knative/conformance/experiment.yaml
kubectl delete -f $ITER8/samples/knative/conformance/baseline.yaml
```

??? info "Understanding what happened"
    1. You created a Knative service with a single revision, sample-app-v1. 
    2. You generated requests for the Knative service using a fortio-job.
    3. You created an Iter8 `Conformance` experiment. In each iteration, Iter8 observed the mean latency, 95th percentile tail-latency, and error-rate metrics collected by Prometheus, and verified that `baseline` satisfied all the objectives specified in the experiment.