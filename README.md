# Keda kafka demo


## Create ns
```bash
oc new-project keda-cma-kafka
```

## Install operators
```bash
oc apply -f Subscription.yaml
```

## Create Keda controller
```bash
oc apply -f KedaController.yaml -n openshift-keda
oc wait kedacontroller/keda --for=condition=Ready --timeout=500s -n openshift-keda
```

## Create kafka cluster
```bash
oc apply -f Kafka.yaml -n keda-cma-kafka
oc wait kafka/demo-cluster --for=condition=Ready --timeout=500s -n keda-cma-kafka
```

## Create kafka consumer
```bash
oc apply -f Build.yaml -n keda-cma-kafka
oc start-build bc/kafka-consumer -n keda-cma-kafka
oc wait build/kafka-consumer-1 --for=condition=Complete --timeout=500s -n keda-cma-kafka
oc apply -f Deployment.yaml -n keda-cma-kafka 
```

## Create kafka producer
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

## Create consumer lag monitor
```bash
# Function args:
# - 1: metrics filter (default kafka_consumergroup_lag)
consumer_lag() {
local metrics_filter=${1:-"^kafka_consumergroup_lag{"}
local pod_name=$(oc get po -l strimzi.io/component-type=kafka-exporter -n keda-cma-kafka --no-headers -o custom-columns=CONTAINER:metadata.name)
oc rsh -n keda-cma-kafka $pod_name curl localhost:9404/metrics | grep -E "$metrics_filter"
}
```

## Produce intial data
```bash
kafka_producer
```

## Verify consumer lag
```bash
consumer_lag
```

## Create ScaledObject
```bash
oc create secret generic keda-kafka-credential -n keda-cma-kafka --from-literal="username=admin" --from-literal="password=$KAFKA_USER_PASSWORD"
oc apply -f ScaledObject.yaml -n keda-cma-kafka
```

## Test

### Setup terminals
screen 1 - operator logs
```bash
oc logs -f -l app=keda-operator -n openshift-keda
```

screen 2 - watch pods
```bash
watch 'oc get po -n keda-cma-kafka -ocustom-columns=CONTAINER:metadata.name,PHASE:.status.phase'
```

screen 3 - watch deploymet
```bash
oc get deployment/kafka-consumer -w -n keda-cma-kafka
```

### Test 1 - produce 1000 messages
screen 4 - produce 1.000 messages
```bash
kafka_producer
```

### Test 2 - produce 1.000.000 messages
screen 4 - produce 1000000 messages
```bash
kafka_producer 1000000 1024
```

### Min replicas 1
screen 4 - patch ScaledObject.spec.minReplicaCount=1
```bash
oc patch scaledobject/kafka-consumer-scaledobject -n keda-cma-kafka --type='json' \
 -p '[{ "op" : "replace", "path" : "/spec/minReplicaCount", "value" : 1 }]'
```

## Docs

Da codice il parametro [activationLagThreshold](https://github.com/kedacore/keda/blob/8a591de4a5d157e067922a89440e19e08d4a0fa6/pkg/scalers/kafka_scaler.go#L959
) viene usata per determinare se effettuare scale down or scale up (0-1 o 1-0), come documentato [qui](https://keda.sh/docs/2.18/concepts/external-scalers/#built-in-scalers-interface):

GetMetricsAndActivity is called on `pollingInterval` and. When activity returns true, KEDA will scale to what is returned by the metric limited by `maxReplicaCount` on the ScaledObject/ScaledJob. When false is returned, KEDA will scale to `minReplicaCount` or optionally `idleReplicaCount`. More details around the defaults and how these options work together can be found on the ScaledObjectSpec.

Mentre il parametro [lagThreshold](https://github.com/kedacore/keda/blob/8a591de4a5d157e067922a89440e19e08d4a0fa6/pkg/scalers/kafka_scaler.go#L909) viene usato per determinate il target value usato definizione del HPA per lo scaler (1-N o N-1), come documentato [qui](https://keda.sh/docs/2.18/concepts/external-scalers/#built-in-scalers-interface):

GetMetricSpecForScaling returns the target value for the HPA definition for the scaler. For more details refer to Implementing [GetMetricSpec](https://keda.sh/docs/2.18/concepts/external-scalers/#5-implementing-getmetricspec).

