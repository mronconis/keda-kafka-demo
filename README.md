# [Demo] Auto-Scaling Kafka Consumers with Custom Metrics Autoscaler

This demo showcases how CMA (Custom Metrics Autoscaler) automatically scales a Kafka consumer application based on consumer lag. As the number of pending messages increases, CMA triggers OpenShift to scale out the consumer pods; when the message load drops, it scales the pods back down to the configured minimum. 

## Overview CMA
The **Custom Metrics Autoscaler** operator is based on the upstream project KEDA. KEDA is an incubating CNCF project, and Red Hat is one of its co-founders, and is an active contributor.

The main **goal** of this autoscaler is to enable autoscaling based on events and custom metrics in a user friendly way. It acts as a thin layer on top of the existing Horizontal Pod Autoscaler, it provides it with external metrics and also manages scaling down to zero.

The Custom Metrics Autoscaler consists of two main components:
- **Operator**: activates and deactivates scalable workloads (Deployments, Stateful Sets,...) by scaling them from and to zero replicas. Operator also manages Custom Resources such as Scaled Objects that define the metadata needed for scaling.
- **Metrics Server**: acts as an adapter to OpenShift API server and provides external metrics to Horizontal Pod Autoscaler Controller which drives autoscaling under the hood.

![Custom Metrics Autoscaler](https://www.redhat.com/rhdc/managed-files/styles/default_800/private/ohc/Custom%20Metrics%20Autoscaler%20on%20OpenShift-3.png.webp?itok=-9ysR0MY)

## Keda Scaler

### Built-in scalers interface
Since external scalers mirror the interface of built-in scalers, it’s worth becoming familiar with the Go `interface` that the built-in scalers implement:
```go
// Scaler interface
type Scaler interface {
	// GetMetricsAndActivity returns the metric values and activity for a metric Name
	GetMetricsAndActivity(ctx context.Context, metricName string) ([]external_metrics.ExternalMetricValue, bool, error)
	// GetMetricSpecForScaling returns the metrics based on which this scaler determines that the ScaleTarget scales. This is used to construct the HPA spec that is created for
	// this scaled object. The labels used should match the selectors used in GetMetrics
	GetMetricSpecForScaling(ctx context.Context) []v2.MetricSpec
	// Close any resources that need disposing when scaler is no longer used or destroyed
	Close(ctx context.Context) error
}
```
For more details see the [documentation](https://keda.sh/docs/2.18/concepts/external-scalers/#built-in-scalers-interface).

### Kafka Scaler implementation
This Kafka Scaler is based on sarama, sarama is a Go client library for Apache Kafka, for details see [Sarama](https://github.com/IBM/sarama).
```go
// GetMetricsAndActivity returns value for a supported metric and an error if there is a problem getting the metric
func (s *kafkaScaler) GetMetricsAndActivity(_ context.Context, metricName string) ([]external_metrics.ExternalMetricValue, bool, error) {
	totalLag, totalLagWithPersistent, err := s.getTotalLag()
	if err != nil {
		return []external_metrics.ExternalMetricValue{}, false, err
	}
	metric := GenerateMetricInMili(metricName, float64(totalLag))

	return []external_metrics.ExternalMetricValue{metric}, totalLagWithPersistent > s.metadata.activationLagThreshold, nil
}
```

When activity returns `true`, KEDA will scale to what is returned by the metric limited by `maxReplicaCount` on the ScaledObject/ScaledJob. When `false` is returned, KEDA will scale to `minReplicaCount` or optionally `idleReplicaCount`. For other details see the full source code at the following [url](https://github.com/kedacore/keda/blob/8a591de4a5d157e067922a89440e19e08d4a0fa6/pkg/scalers/kafka_scaler.go#L952).
```go
func (s *kafkaScaler) GetMetricSpecForScaling(context.Context) []v2.MetricSpec {
	var metricName string
	if s.metadata.topic != "" {
		metricName = fmt.Sprintf("kafka-%s", s.metadata.topic)
	} else {
		metricName = fmt.Sprintf("kafka-%s-topics", s.metadata.group)
	}

	externalMetric := &v2.ExternalMetricSource{
		Metric: v2.MetricIdentifier{
			Name: GenerateMetricNameWithIndex(s.metadata.triggerIndex, kedautil.NormalizeString(metricName)),
		},
		Target: GetMetricTarget(s.metricType, s.metadata.lagThreshold),
	}
	metricSpec := v2.MetricSpec{External: externalMetric, Type: kafkaMetricType}
	return []v2.MetricSpec{metricSpec}
}
```
This function returns the target value for the HPA definition for the scaler. For other details see the full source code at the following [url](https://github.com/kedacore/keda/blob/8a591de4a5d157e067922a89440e19e08d4a0fa6/pkg/scalers/kafka_scaler.go#L897).

## Setup environment

### Create namespace
```bash
oc new-project keda-cma-kafka
```

### Install operators
```bash
oc apply -f Subscription.yaml
```

### Create Keda controller
```bash
oc apply -f KedaController.yaml -n openshift-keda
oc wait kedacontroller/keda --for=condition=Ready --timeout=500s -n openshift-keda
```

### Create Kafka cluster
```bash
oc apply -f Kafka.yaml -n keda-cma-kafka
oc wait kafka/demo-cluster --for=condition=Ready --timeout=500s -n keda-cma-kafka
```

### Create Kafka consumer
```bash
oc apply -f Build.yaml -n keda-cma-kafka
oc start-build bc/kafka-consumer -n keda-cma-kafka
oc wait build/kafka-consumer-1 --for=condition=Complete --timeout=500s -n keda-cma-kafka
oc apply -f Deployment.yaml -n keda-cma-kafka 
```

### Create Kafka producer
```bash
# Function args:
# - 1: number of records (default 1000)
# - 2: network throughput (default -1=no limit)
# - 3: record size (default 100 bytes)
kafka_producer() {
local kafka_user_pwd=$(oc get secret admin -n keda-cma-kafka -o jsonpath='{.data.password}' | base64 -d)
local num_records=${1:-"1000"}
local throughput=${2:-"-1"}
local record_size=${3:-"100"}
oc run kafka-producer -ti --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --rm=true --restart=Never -n keda-cma-kafka -- /bin/bash -c "cat >/tmp/client.properties <<EOF 
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username=admin password=$kafka_user_pwd;
EOF
bin/kafka-producer-perf-test.sh --producer-props bootstrap.servers=demo-cluster-kafka-bootstrap.keda-cma-kafka.svc.cluster.local:9092 --topic test-in --record-size $record_size --num-records $num_records --producer.config=/tmp/client.properties --throughput $throughput" 2>/dev/null
}
```

### Create consumer lag monitor
```bash
# Function args:
# - 1: metrics filter (default kafka_consumergroup_lag)
consumer_lag() {
local metrics_filter=${1:-"^kafka_consumergroup_lag{"}
local pod_name=$(oc get po -l strimzi.io/component-type=kafka-exporter -n keda-cma-kafka --no-headers -o custom-columns=CONTAINER:metadata.name)
oc rsh -n keda-cma-kafka $pod_name curl localhost:9404/metrics | grep -E "$metrics_filter"
}
```

### Produce intial data
```bash
kafka_producer
```

### Verify consumer lag
```bash
consumer_lag
```

### Create ScaledObject
```bash
oc create secret generic keda-kafka-credential -n keda-cma-kafka --from-literal="username=admin" --from-literal="password=$KAFKA_USER_PASSWORD"
oc apply -f ScaledObject.yaml -n keda-cma-kafka
```

## Demo

### Test case 1 - Activate and deactivate Scaler

In this scenario, KEDA is expected to activate the workload and, once processing is complete (when totalLag is less than or equal to the activationLagThreshold), deactivate the deployment.
```bash
# produces 20 messages with a throughput of 5 messages per second.
kafka_producer 200 5
```

### Test case 2 - Scale up and scale down to zero

In this scenario, KEDA is expected to activate the deployment and scale up replicas. Once processing is complete (when totalLag is less than or equal to the lagThreshold), it will scale back down to zero as defined by `minReplicaCount`.
```bash
# produces 1.000.000 messages with a throughput of 10.000 messages per second.
kafka_producer 1000000 10000
```

### Test case 3 - Scale up and scale down by HPA

Configure ScaledObject to have `minReplicaCount=1` (equal to always active):
```bash
# patch ScaledObject.spec.minReplicaCount=1
oc patch scaledobject/kafka-consumer-scaledobject -n keda-cma-kafka --type='json' \
 -p '[{ "op" : "replace", "path" : "/spec/minReplicaCount", "value" : 1 }]'
```

In the following scenario, the expected behavior is that the replicas scale up during processing and scale back down once processing is complete, in accordance with the HPA configuration.
```bash
#produces 1.000.000 messages with a throughput of 10.000 messages per second.
kafka_producer 1000000 10000
```

### Test case 4 - Invalid consumer offset

TBD.