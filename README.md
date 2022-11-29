Patrick's HPA sandbox session.

Many customer come to us with HPA tickets. Public docs on this can be quite confusing (trust me I know) which is the reason for this session :D 

Following the public docs on HPA: https://docs.datadoghq.com/containers/cluster_agent/external_metrics/?tab=helm#overview

Assuming that your cluster agent is deployed already via Helm.

1. Your app key is required, either set in the values.yaml or passed in the helm command as such:
```
helm upgrade <RELEASE_NAME> -f values.yaml --set datadog.apiKey=<API_KEY> --set datadog.appKey=<APP_KEY> datadog/datadog
```

Ensure in your helm chart that this is set to enable HPA:
```
clusterAgent:
  enabled: true
  metricsProvider:
    enabled: true
    # clusterAgent.metricsProvider.useDatadogMetrics
    # Enable usage of DatadogMetric CRD to autoscale on arbitrary Datadog queries
    useDatadogMetrics: true
```

Must use an existing metric in your sandbox. In my case I am using kubernetes.pods.running

2. An HPA manifest and DatadogMetric manifest needs to be created.

In the HPA manifest:
```
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: kubetest
spec:
  minReplicas: 1
  maxReplicas: 3
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa
  metrics:
  - type: External
    external:
      metricName: datadogmetric@default:test-metric
      targetAverageValue: 9
```

Following this template:
```
  metrics:
    - type: External
      external:
        metricName: "datadogmetric@<namespace>:<datadogmetric_name>"
        targetAverageValue: #required
```
The targetAverageValue is required but for the purposes of the session, this doesn't need to be a specific value.

In metricName set the namespace to default since we are testing this in the default namespace.

In metricName set the datadogmetric_name to match the name set in DatadogMetric manifest as shown below.

In the DatadogMetric manifest:
```
apiVersion: datadoghq.com/v1alpha1
kind: DatadogMetric
metadata:
  name: test-metric
spec:
  query: avg:kubernetes.pods.running{*}
```

3. After deploying these manifests and waiting for a short moment, run to confirm that they are deployed successfully:

Run:
```
kubectl get hpa
```
```
NAME       REFERENCE        TARGETS             MINPODS   MAXPODS   REPLICAS   AGE
kubetest   Deployment/hpa   <unknown>/9 (avg)   1         3         0          30m
```

Run:
```
kubectl get DatadogMetric
```
```
NAME                                                ACTIVE   VALID   VALUE                REFERENCES             UPDATE TIME
test-metric                                         True     True    1.4615384615384615   hpa:default/kubetest   63s
```

4. Then run a describe on the DatadogMetric:
```
kubectl describe DatadogMetric test-metric
```

The status section of the output should be as follows:
```
Status:
  Autoscaler References:  hpa:default/kubetest
  Conditions:
    Last Transition Time:  2022-11-29T21:03:53Z
    Last Update Time:      2022-11-29T21:25:32Z
    Status:                True
    Type:                  Active
    Last Transition Time:  2022-11-29T21:04:12Z
    Last Update Time:      2022-11-29T21:25:32Z
    Status:                True
    Type:                  Valid
    Last Transition Time:  2022-11-29T21:03:53Z
    Last Update Time:      2022-11-29T21:25:32Z
    Status:                True
    Type:                  Updated
    Last Transition Time:  2022-11-29T21:03:53Z
    Last Update Time:      2022-11-29T21:25:32Z
    Status:                False
    Type:                  Error
  Current Value:           1.4615384615384615
Events:                    <none>
```
The error status should be false and the rest should be true.

Also Current Value should not be 0.

If the above is satisfied, you have successfully enabled the Custom Metrics Server.