# Spark Streaming Performance Testing Report

## Overview

This document outlines our performance testing of a Spark Streaming application on Kubernetes. We conducted multiple tests to identify bottlenecks, optimize configurations, and achieve high throughput processing of data from Kafka sources.

## Test Environment

- **Platform**: Spark on Kubernetes
- **Data Source**: Kafka topics
- **Data Generator**: Python-based Kafka producer with configurable throughput
- **Monitoring**: Prometheus & Grafana
- **Storage**: S3 for checkpointing

## Initial Setup

Our first goal was to establish a baseline performance. We set up the following components:
- Spark on Kubernetes environment
- Kafka as the streaming source
- Producer application for simulating variable load patterns
- Prometheus and Grafana for monitoring

## Test Journey and Findings

### Test 1: Initial Performance

Our first test revealed extremely low throughput:

**Initial Results:**
- **Rate of ingestion:** ~18.5 keys/second (~1,107 keys/minute)
- **Time to ingest 100,000 keys:** ~90 minutes

**Initial Configuration:**
```yaml
processing_engine: 'pyspark_k8s'
processing_engine_configs:
  deploy-mode: 'cluster'
  spark.kubernetes.driver.memory: '5000m'
  spark.kubernetes.executor.memory: '5000m'
  spark.kubernetes.authenticate.driver.serviceAccountName: 'spark-operator'
  spark.kubernetes.file.upload.path: '/opt/spark/work-dir'
  spark.kubernetes.namespace: 'spark-streaming-jobs'
  num-executors: 2
  spark.executor.cores: 2
  spark.driver.cores: 1
  spark.kubernetes.driver.limit.cores: '5000m'
  spark.kubernetes.executor.limit.cores: '5000m'
  spark.executor.instances: 5
```

### Test 2: Identifying and Resolving the Bottleneck

We discovered a critical bottleneck in our Redis ingestion pipeline:
- Each record was processed individually with a new Redis connection
- This connection overhead severely limited throughput

**After fixing the Redis connection issue:**
- **Rate of ingestion:** ~3,866 keys/second (~232,001 keys/minute)
- **Projected time to ingest:**
  - 100,000 keys: ~26 seconds
  - 1,000,000 keys: ~4.3 minutes
  - 10,000,000 keys: ~43.1 minutes

**Updated Configuration:**
```yaml
processing_engine: 'pyspark_k8s'
processing_engine_configs:
  deploy-mode: 'cluster'
  spark.kubernetes.driver.memory: '10000m'
  spark.kubernetes.executor.memory: '10000m'
  spark.kubernetes.authenticate.driver.serviceAccountName: 'spark-operator-spark'
  spark.hadoop.fs.s3a.aws.credentials.provider: 'com.amazonaws.auth.WebIdentityTokenCredentialsProvider'
  spark.kubernetes.container.image.pullSecrets: 'docker-secret-cred'
  spark.kubernetes.file.upload.path: '/opt/spark/work-dir'
  spark.kubernetes.namespace: 'spark-streaming-jobs'
  num-executors: 2
  spark.executor.cores: 1
  spark.driver.cores: 1
  spark.kubernetes.driver.limit.cores: '10000m'
  spark.kubernetes.executor.limit.cores: '10000m'
  spark.executor.instances: 5
```

### Test 3: Optimizing Executor Configuration

We tested various executor configurations to find the optimal setup:

```yaml
processing_engine: 'pyspark_k8s'
processing_engine_configs:
  spark.kubernetes.driver.memory: '5000m'
  spark.kubernetes.executor.memory: '5000m'
  num-executors: 6
  spark.executor.cores: 2
  spark.driver.cores: 1
  spark.kubernetes.driver.limit.cores: '5000m'
  spark.kubernetes.executor.limit.cores: '5000m'
  spark.executor.instances: 6
```

**Results:**
- 5 executor pods running, 1 in pending state
- Processed 1 million records in 2:40 minutes
- Processed 2 million records in 4:30 minutes
- Processed 5 million records in 10:05 minutes
- Completed 48 batches in 10 minutes with no failures

**Key Findings:**
- **2 cores** per executor performed better than **1 core**
- Number of executors had the greatest impact on performance
- Tests with **>10GB** executor memory failed due to insufficient cluster memory
- Lower memory configurations (**3-5GB**) were more stable for pod scheduling
- Fastest configuration to reach 1M records: 2:40 minutes

### Test 4: Complex Data Schema Testing

We increased schema complexity by adding more fields and nested structures to test the impact on processing performance.

### Test 5: Scaling Infrastructure

We expanded our Kubernetes node groups, adding a new t3.xlarge node group to the existing spot instance.

### Test 6: Multiple Kafka Producers

To achieve higher throughput, we implemented multiple Kafka producers to increase the data ingestion rate:

**Findings:**
- Performance improved significantly
- However, the job became unstable after ~20 minutes due to excessive load
- Micro-batch processing times increased exponentially

### Test 7: Stabilizing Performance with Additional Resources

We added an on-demand m5a.4xlarge node group:

**Results:**
- Basic logic stabilized at 2:40 minutes for 1M records
- Micro-batch processing times remained under 30 seconds
- Medium and complex logic still exhibited instability

### Test 8: Dynamic Allocation for Stabilization

We implemented dynamic allocation to handle varying loads:

```yaml
spark.dynamicAllocation.enabled: "true"
spark.dynamicAllocation.minExecutors: "2"
spark.dynamicAllocation.initialExecutors: "16"
spark.dynamicAllocation.maxExecutors: "18"
spark.dynamicAllocation.shuffleTracking.enabled: "true"
spark.dynamicAllocation.executorIdleTimeout: "60s"
```

We then tested three load patterns:
1. **Steady Load**: Fixed number of Kafka producers
2. **Gradual Load**: Incrementally increasing producers
3. **Spike Load**: Sudden increases and decreases in producers

After further testing with modified configs:

```yaml
spark.dynamicAllocation.enabled: "true"
spark.dynamicAllocation.minExecutors: "3"
spark.dynamicAllocation.initialExecutors: "5"
spark.dynamicAllocation.maxExecutors: "20"
spark.dynamicAllocation.executorIdleTimeout: "30s"
spark.dynamicAllocation.schedulerBacklogTimeout: "1s"
spark.streaming.kafka.maxRatePerPartition: "12000"
spark.streaming.backpressure.enabled: "true"
```

We successfully reduced medium complexity logic processing time from 10 minutes to 2:40 minutes for 1M records by using 8 Kafka producers.

## Final Optimized Configuration

After extensive testing, our final configuration for handling the most complex logic:

```yaml
processing_engine: 'pyspark_k8s'
processing_engine_configs:
  deploy-mode: 'cluster'
  spark.kubernetes.driver.memory: '24000m'
  spark.kubernetes.executor.memory: '24000m'
  spark.kubernetes.authenticate.driver.serviceAccountName: 'spark-operator'
  spark.kubernetes.file.upload.path: '/opt/spark/work-dir'
  spark.kubernetes.namespace: 'spark-streaming-jobs'
  num-executors: 1
  spark.executor.cores: 12
  spark.driver.cores: 10
  spark.kubernetes.driver.limit.cores: '24000m'
  spark.kubernetes.executor.limit.cores: '24000m'
  spark.executor.instances: 1
```

**Final Achievement:** 1.2 billion records processed per hour with our most complex logic.

## Summary of Key Findings

| Optimization                     | Impact                                               |
|----------------------------------|------------------------------------------------------|
| Redis connection pooling         | Improved from 18.5 to 3,866 keys/second              |
| 2 cores vs 1 core per executor   | Significant performance improvement                  |
| Memory optimization (3-5GB)      | Better pod scheduling reliability                    |
| Multiple Kafka producers         | Higher throughput but potential instability          |
| Dynamic allocation               | Better handling of variable loads                    |
| Resource scaling                 | Larger instance types stabilized complex workloads   |

## Recommendations

1. **Connection Pooling**: Ensure proper connection pooling for any external system interactions
2. **Multi-producer Approach**: Use multiple Kafka producers for high-volume testing
3. **Dynamic Allocation**: Enable dynamic allocation with appropriate timeouts based on workload patterns
4. **Resource Optimization**: Balance between executor count and cores per executor
5. **Memory Configuration**: Keep memory allocations reasonable for your cluster capacity
6. **Backpressure Management**: Enable backpressure for high-volume scenarios
7. **Load Testing Patterns**: Test with steady, gradual, and spike load patterns
8. **Long-running Tests**: Run tests for extended periods (hours) to identify stability issues
9. **Monitoring**: Implement comprehensive monitoring for Spark metrics, especially micro-batch processing times

By implementing these optimizations, we achieved a stable system capable of processing 1.2 billion records per hour with complex transformation logic.