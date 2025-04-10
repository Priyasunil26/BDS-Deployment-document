# Kafka-KRaft Setup Guide

This document provides a step-by-step guide to setting up Kafka with KRaft mode using Kubernetes and Helm. Each step includes explanations to clarify its purpose and function.

## 1. Create Namespace

**Command:**
```bash
kubectl create namespace kafka-kraft
```
**Explanation:**
This command creates a Kubernetes namespace named `kafka-kraft`, which helps in logically separating Kafka resources from other services within the same cluster.

## 2. Add and Update Helm Repository

**Commands:**
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

**Explanation:**
- The first command adds the Bitnami Helm chart repository which includes pre-configured Kafka charts.
- The second command updates the local cache with the latest available chart versions from Bitnami.

---

## 3. Install Kafka-KRaft (Controller and Broker)

**File: `kafka-kraft.yaml`**
```yaml
controller:
  replicaCount: 1
  configurationOverrides:
    node.id: "1"
    log.dirs: "/bitnami/kafka/data"
    listeners: "PLAINTEXT://:9092"
    advertised.listeners: "PLAINTEXT://kafka-kraft.kafka.svc.cluster.local:9092"
    process.roles: "controller,broker"
    controller.quorum.voters: "1@kafka-0.kafka-headless.kafka-kraft.svc.cluster.local:9092,2@kafka-1.kafka-headless.kafka-kraft.svc.cluster.local:9092,3@kafka-2.kafka-headless.kafka-kraft.svc.cluster.local:9092"
```

**Command:**
```bash
helm install kafka-kraft bitnami/kafka --namespace kafka-kraft -f kafka-kraft.yaml
```

**Explanation:**
- Deploys the Kafka controller in KRaft mode using the custom configuration.
- Specifies `node.id`, data directories, listener configuration, roles, and controller quorum voters for consensus.

**Additional Command (for brokers):**
```bash
helm upgrade --install kafka-broker oci://registry-1.docker.io/bitnamicharts/kafka \
  --namespace kafka-kraft \
  --set controller.enabled=false \
  --set replicaCount=3 \
  --set kraft.enabled=true \
  --set kraft.controllers=3 \
  --set service.type=ClusterIP \
  --set allowAnonymousLogin=true
```

**Explanation:**
- Installs brokers with KRaft mode enabled.
- Disables controller in this deployment, sets up 3 broker replicas, and allows anonymous login for easier access (not recommended for production).

---

## 4. Verify Configuration

**Command:**
```bash
kubectl exec -it -n kafka-kraft kafka-broker-controller-0 -- cat /opt/bitnami/kafka/config/server.properties
```

**Explanation:**
Verifies that Kafka has applied the intended configuration by checking the contents of the server properties file inside the running pod.

---

## 5. Configure Authentication

**File: `client.properties`**
```properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="user1" password="etet92dP92";
```

**Command:**
```bash
kubectl create configmap kafka-client-config --from-file=client.properties -n kafka-kraft
```

**Explanation:**
Creates a ConfigMap to store authentication credentials securely in Kubernetes, allowing Kafka clients to authenticate using SASL.

---

## 6. Mount ConfigMap in StatefulSets

**Configuration Snippet:**
```yaml
spec:
  template:
    spec:
      volumes:
        - name: kafka-client-config
          configMap:
            name: kafka-client-config
      containers:
        - name: kafka
          volumeMounts:
            - name: kafka-client-config
              mountPath: /opt/bitnami/kafka/config/client.properties
              subPath: client.properties
```

**Commands:**
```bash
kubectl edit statefulset kafka-broker-controller -n kafka-kraft
kubectl edit statefulset kafka-kraft-controller -n kafka-kraft
kubectl rollout restart statefulset -n kafka-kraft
```

**Explanation:**
- Edits StatefulSets to mount the ConfigMap containing client authentication details.
- Restarts the StatefulSets to apply changes.

---

## 7. Test Kafka-KRaft Deployment

**Create Topic:**
```bash
kubectl exec -it -n kafka-kraft kafka-broker-controller-0 -- \
  kafka-topics.sh --create --topic my-test-topic --partitions 3 --replication-factor 1 \
  --bootstrap-server kafka-kraft-controller-0.kafka-broker-controller-headless.kafka-kraft.svc.cluster.local:9092 \
  --command-config /opt/bitnami/kafka/config/client.properties
```

**List Topics:**
```bash
kubectl exec -it -n kafka-kraft kafka-broker-controller-0 -- \
  kafka-topics.sh --list \
  --bootstrap-server kafka-kraft-controller-0.kafka-broker-controller-headless.kafka-kraft.svc.cluster.local:9092 \
  --command-config /opt/bitnami/kafka/config/client.properties
```

**Explanation:**
- Verifies Kafka functionality by creating and listing topics.
- Uses SASL authentication for secure communication.

---

## 8. Additional Configuration and Troubleshooting

**Commands:**
```bash
kubectl logs -n kafka-kraft statefulset/kafka-kraft-controller
kubectl scale statefulset kafka-broker-controller --replicas=5 -n kafka-kraft
```

**Explanation:**
- The first command helps monitor the logs for troubleshooting.
- The second command scales Kafka broker replicas to increase fault tolerance and throughput.

---

By following this guide, you’ll have a functional Kafka cluster operating in KRaft mode with authentication and scalability built-in. This setup is suitable for development and can be extended for production with enhanced security and persistence.

