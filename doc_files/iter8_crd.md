# iter8 `Experiment` CRD

When iter8 is installed, a new Kubernetes CRD is added to your cluster. Our CRD kind and current API version are as follows:

```yaml
apiVersion: iter8.tools/v1alpha2
kind: Experiment
```

Below we document iter8's Experiment CRD. For clarity, we break the documentation down into the CRD's 3 sections: `spec`, `metrics`, and `status`.

## `Experiment spec`

Following the Kubernetes model, the `spec` section specifies the details of the object and its desired state. The `spec` of an `Experiment` custom resource identifies the target service of a candidate release or A/B test, the baseline deployment corresponding to the stable service version, the candidate deployments corresponding to the service versions being assessed, etc. In the YAML representation below, we show sample values for the `spec` attributes and comments describing their meaning and whether or not they are optional.

At a high level, the `spec` contains the following sections, each of which is described in more detail below.

```yaml
spec:
    service: # [required] details of the service to be tested including the baseline and candidate versions
    routingReference: # [optional] reference to an existing istio VirtualService that should be used for the service
    criteria: # [optional] list of assessment criteria used to determine value of candidate version(s)
    duration: # [optional] defines the amount of time an experiment will take to complete
    trafficControl: # [optional] specifies how traffic will be managed during the experiment
    cleanup: # [optional] specifies whether or not some versions of the service will be deleted at the end of the experiment
    analyticsServiceURL: # [optional] the URL of the analytics service
    manualOverride: # [optional] identifies manual actions that can be taken to override automated behavior
    metrics: # [optional] List of metrics definitions
```

### `spec.service`

The `service` identifies the service to which the experiment applies. Further, it identifies the baseline version and one or more candidate versions. For a canary experiment, only 1 candidate should be specified. A/B tests can specify any number of candidates. It is assumed that all versions are executing in the same namespace of the same cluster.

### `spec.routingReference`

A `routingReference` is a kubernetes `OwnerReference` to an existing Istio `VirtualService` already defined for the Kubernetes `Service`. iter8 will reuse the `VirtualService` instead of creating a new one.

### `spec.criteria`

A list of criteria can be specified that should be used to assess the relative quality of the candidates with respect to each other and to the baseline. Each criterion references a _metric_ and specifies either what absolute values are acceptable or how different the value may be from the baseline version. If any one critieron is violated by a version, that version will be deemed unsuitable. If the flag `cutoffTrafficOnViolation` is set, action to block traffic to the failing version will be taken immediately.
One criterion can be identified as a _reward_. Rewards are used in A/B testing to evalutate the most valuable (desired) version.

Some example criteria are shown here:

```yaml
criteria:
  - metric: iter8_mean_latency
      threshold: # (optional section)
        type: relative # can be relative or absolute
        value: 1.1 # float
    - metric: iter8_error_rate
    - metric: iter8_error_count
      threshold:
        type: absolute
        value: 10
        cutoffTrafficOnViolation: true # if the threshold is violated, the candidate violating the threshold will not get any traffic (optional; default = false)
    - metric: conversion_rate
        isReward: true # this is a reward metric. Only ratio metrics can be reward metrics. There can be at most one reward metric (optional; default = false)
```

### `spec.duration`

The `duration` options specify how long an experiment will run. The total run time is the product of the time per iteration and the number of iterations.

```yaml
duration:
    interval: 30s # [optional] length of each iteration. Specified using standard go time specification. Default value is 30s.
    maxIterations: 100 # [optional] the maximum number of iterations an experiment lasts. Default is 100.
```

### `spec.trafficControl`

The `trafficControl` options allow for greater control the distribution of traffic between versions of the service under test. These are documented in the example below. For further details about the traffic control choices, see URL. 

```yaml
trafficControl: # [optional]
    percentage: 80 # [optional] Specifies the total amount of traffic that should be used for experimental purposes. The remaining traffic will all be directed at the baseline version. Def
    maxIncrement: 2 # [optional] A maximum percentage by which the traffic distribution can change in a given iteration of the experiment. Default is 2.
    strategy: progressive # [optional] Choice of algorithm to be used to determine recommended traffic split. Value options are "progressive" (default), "top_2", and "uniform". For details see: URL
    onTermination: to_winner # [optional] Indicates how traffic should be distributed between versions when the experiment ends (whether successfully or not). Valid values are "to_winner", "to_baseline" and "keep_last". The efault is "to_winner". The specification here can be overridden if terminated manually, see manualOverrides.trafficSplit.
```

### `spec.cleanup`

When set to `true`, `cleanup` indicates that when the experiment ends (whether successful or not), any versions of the service not receiving traffic should be deleted. `cleanup` defaults to `false`.

```yaml
cleanup: true # [optional] default is false
```

### `spec.analyticsServiceURL`

### `spec.manualOverride`

While an experiment is executing, the `Experiment` can be patched to add a `manualOverride`. Manual actions are taken immediately and have precedence over the rest of the experiment specification. They only affect on experiments that are progressing; they have no effect on experiments that are completed. A running experiment can be _paused_, a paused experiment can be _resumed_, and any experiment can be _terminated_. When pausing, no change is made the traffic split between versions. While paused, no updates to the assessment criteria are made. When terminating an experiment, one may optionally specify a final traffic distribution between the versions of the service. Specified in integer quantities, it should sum to 100%.

```yaml
manualOverride: # [optional]
    action: terminate # [required] Identifies what action should be taken. Valid values are "pause", "resume", and "terminate".

    trafficSplit: # [optional] specifies how traffic should be distributed between the service versions.Applies only when the action is terminate. Specified as integer values, they must add to 100.
        reviews-v2: 10
        reviews-v3: 90
```

### `spec.metrics`

Information about all Prometheus metrics known to iter8 are stored in a Kubernetes `ConfigMap` named _`iter8_metrics`_. When iter8 is installed, that `ConfigMap` is populated with information on the 3 metrics that iter8 supports out of the box, namely: `iter8_latency`, `iter8_error_rate`, and `iter8_error_count`. Users can add their own custom metrics.

When an `Experiment` custom resource is created, the iter8 controller will check the metric names referenced by `.spec.analysis.successCriteria`, look them up in the `ConfigMap`, retrieve the information about them from the `ConfigMap`, and store that information in the `metrics` section of the newly created `Experiment` object. The information about a metric allows the iter8 analytics service to query Prometheus to retrieve metric values for the baseline version and candidate versions of the service . Below we show an example of how a metric is stored in an `Experiment` object.

```yaml
metrics:
  iter8_latency:
    absent_value: None
    is_counter: false
    query_template: (sum(increase(istio_request_duration_seconds_sum{source_workload_namespace!='knative-serving',reporter='source'}[$interval]$offset_str))
      by ($entity_labels)) / (sum(increase(istio_request_duration_seconds_count{source_workload_namespace!='knative-serving',reporter='source'}[$interval]$offset_str))
      by ($entity_labels))
    sample_size_template: sum(increase(istio_requests_total{source_workload_namespace!='knative-serving',reporter='source'}[$interval]$offset_str))
      by ($entity_labels)
```

## `Experiment status`

Following the Kubernetes model, the `status` section contains all relevant runtime details pertaining to the `Experiment` custom resource. In the YAML representation below, we show sample values for the `status` attributes and comments describing their meaning.

```yaml
  status:
    # the last analysis state
    analysisState: {}

    # assessment returned from the analytics service
    assessment:
      conclusions:
      - The experiment needs to be aborted
      - All success criteria were not met

    # list of boolean conditions describing the status of the experiment
    # for each condition, if the status is "False", the reason field will give detailed explanations
    # lastTransitionTime records the time when the last change happened to the corresponding condition
    # when a condition is not set, its status will be "Unknown"
    conditions:

    # AnalyticsServiceNormal is "True" when the controller can get an interpretable response from the analytics service
    - lastTransitionTime: "2019-12-20T05:38:37Z"
      status: "True"
      type: AnalyticsServiceNormal

    # ExperimentCompleted tells whether the experiment is completed or not
    - lastTransitionTime: "2019-12-20T05:39:37Z"
      status: "True"
      type: ExperimentCompleted

    # ExperimentSucceeded indicates whether the experiment succeeded or not when it is completed
    - lastTransitionTime: "2019-12-20T05:39:37Z"
      message: Aborted
      reason: ExperimentFailed
      status: "False"
      type: ExperimentSucceeded

    # MetricsSynced states whether the referenced metrics have been retrieved from the ConfigMap and stored in the metrics section
    - lastTransitionTime: "2019-12-20T05:38:22Z"
      status: "True"
      type: MetricsSynced

    # Ready records the status of the latest-updated condition
    - lastTransitionTime: "2019-12-20T05:39:37Z"
      message: Aborted
      reason: ExperimentFailed
      status: "False"
      type: Ready

    # RoutingRulesReady indicates whether the routing rules are successfully created/updated
    - lastTransitionTime: "2019-12-20T05:38:22Z"
      tatus: "True"
      type: RoutingRulesReady

    # TargetsProvided is "True" when both the baseline and the candidate versions of the targetService are detected by the controller; otherwise, missing elements will be shown in the reason field
    - lastTransitionTime: "2019-12-20T05:38:37Z"
      status: "True"
      type: TargetsProvided

    # the current experiment's iteration
    currentIteration: 2

    # Unix timestamp in milliseconds corresponding to when the experiment started
    startTimestamp: "1576820317351"

    # Unix timestamp in milliseconds corresponding to when the experiment finished
    endTimestamp: "1576820377696"

    # The url to he Grafana dashboard pertaining to this experiment
    grafanaURL: http://localhost:3000/d/eXPEaNnZz/iter8-application-metrics?var-namespace=bookinfo-iter8&var-service=reviews&var-baseline=reviews-v3&var-candidate=reviews-v5&from=1576820317351&to=1576820377696

    # the time when the previous iteration was completed
    lastIncrementTime: "2019-12-20T05:39:07Z"

    # this is the message to be shown in the STATUS column for the `kubectl` printer, which summarizes the experiment situation
    message: 'ExperimentFailed: Aborted'

    # the experiment's current phase 
    # values could be: Initializing, Progressing, Pause, Completed
    phase: Completed

    # the current traffic split
    trafficSplitPercentage:
      baseline: 100
      candidate: 0
  ```
