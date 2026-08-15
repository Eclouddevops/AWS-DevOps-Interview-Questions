# AWS RabbitMQ & Redis (ElastiCache) - Step-by-Step Configuration Guide

## Table of Contents
1. [Amazon MQ (RabbitMQ) Setup](#1-amazon-mq-rabbitmq-setup)
2. [RabbitMQ Cluster Configuration](#2-rabbitmq-cluster-configuration)
3. [Queue & Exchange Design](#3-queue--exchange-design)
4. [ElastiCache Redis Setup](#4-elasticache-redis-setup)
5. [Redis Cluster Mode Configuration](#5-redis-cluster-mode-configuration)
6. [Caching Patterns & Implementation](#6-caching-patterns--implementation)
7. [Monitoring & Alarms](#7-monitoring--alarms)
8. [Scaling & High Availability](#8-scaling--high-availability)
9. [Backup & Recovery](#9-backup--recovery)
10. [Production Best Practices](#10-production-best-practices)

---

## 1. Amazon MQ (RabbitMQ) Setup

### Step 1.1: Create RabbitMQ Broker


```bash
# Create Multi-AZ RabbitMQ cluster
aws mq create-broker \
  --broker-name production-rabbitmq \
  --engine-type RABBITMQ \
  --engine-version 3.11.20 \
  --deployment-mode CLUSTER_MULTI_AZ \
  --host-instance-type mq.m5.large \
  --publicly-accessible false \
  --subnet-ids subnet-private-az1 subnet-private-az2 subnet-private-az3 \
  --security-groups sg-rabbitmq123456 \
  --users '[{
    "Username": "admin",
    "Password": "SecureP@ssw0rd!",
    "ConsoleAccess": true
  }, {
    "Username": "app_user",
    "Password": "AppSecureP@ss!",
    "ConsoleAccess": false
  }]' \
  --auto-minor-version-upgrade \
  --maintenance-window-start-time '{"DayOfWeek": "SUNDAY", "TimeOfDay": "04:00", "TimeZone": "UTC"}' \
  --logs '{"General": true, "Audit": true}' \
  --encryption-options '{"UseAwsOwnedKey": false, "KmsKeyId": "arn:aws:kms:us-east-1:123456789012:key/xxx"}' \
  --tags Environment=production Team=platform

# Get broker details
aws mq describe-broker --broker-id b-xxxx-yyyy-zzzz
```

### Step 1.2: Security Group for RabbitMQ

```bash
aws ec2 create-security-group \
  --group-name production-rabbitmq-sg \
  --description "RabbitMQ Production - AMQP and Management" \
  --vpc-id vpc-0123456789abcdef0

# AMQP port (from application)
aws ec2 authorize-security-group-ingress \
  --group-id sg-rabbitmq123456 \
  --protocol tcp --port 5671 \
  --source-group sg-app123456

# Management console (from bastion/VPN only)
aws ec2 authorize-security-group-ingress \
  --group-id sg-rabbitmq123456 \
  --protocol tcp --port 443 \
  --source-group sg-bastion123456
```

---

## 2. RabbitMQ Cluster Configuration

### Step 2.1: Connection Setup (Python)

```python
import pika
import ssl

# Production connection with TLS
ssl_context = ssl.create_default_context()
ssl_options = pika.SSLOptions(ssl_context, "b-xxxx.mq.us-east-1.amazonaws.com")

credentials = pika.PlainCredentials('app_user', 'AppSecureP@ss!')
parameters = pika.ConnectionParameters(
    host='b-xxxx.mq.us-east-1.amazonaws.com',
    port=5671,
    virtual_host='/',
    credentials=credentials,
    ssl_options=ssl_options,
    heartbeat=60,
    blocked_connection_timeout=300,
    connection_attempts=3,
    retry_delay=5
)

connection = pika.BlockingConnection(parameters)
channel = connection.channel()
```

### Step 2.2: Declare Exchanges and Queues

```python
# Declare durable exchange
channel.exchange_declare(
    exchange='orders',
    exchange_type='topic',
    durable=True
)

# Declare quorum queue (recommended for durability)
channel.queue_declare(
    queue='order-processing',
    durable=True,
    arguments={
        'x-queue-type': 'quorum',           # Quorum queue for HA
        'x-delivery-limit': 3,              # Max retries
        'x-dead-letter-exchange': 'dlx',    # Dead letter exchange
        'x-dead-letter-routing-key': 'order-processing.dlq',
        'x-message-ttl': 60000,             # 60 second TTL
        'x-max-length': 100000              # Max queue depth
    }
)

# Bind queue to exchange
channel.queue_bind(
    queue='order-processing',
    exchange='orders',
    routing_key='order.created.*'
)

# Dead letter queue
channel.exchange_declare(exchange='dlx', exchange_type='direct', durable=True)
channel.queue_declare(queue='order-processing-dlq', durable=True)
channel.queue_bind(queue='order-processing-dlq', exchange='dlx', routing_key='order-processing.dlq')
```

---

## 3. Queue & Exchange Design

### Step 3.1: Publisher Implementation

```python
import json
import pika
from datetime import datetime

class MessagePublisher:
    def __init__(self, connection_params):
        self.connection = pika.BlockingConnection(connection_params)
        self.channel = self.connection.channel()
        self.channel.confirm_delivery()  # Publisher confirms

    def publish_order(self, order_data):
        message = json.dumps({
            'order_id': order_data['id'],
            'event': 'order.created',
            'timestamp': datetime.utcnow().isoformat(),
            'data': order_data
        })

        try:
            self.channel.basic_publish(
                exchange='orders',
                routing_key=f"order.created.{order_data['region']}",
                body=message,
                properties=pika.BasicProperties(
                    delivery_mode=2,          # Persistent message
                    content_type='application/json',
                    message_id=str(order_data['id']),
                    timestamp=int(datetime.utcnow().timestamp()),
                    headers={'retry_count': 0}
                ),
                mandatory=True  # Ensure message is routed
            )
            return True
        except pika.exceptions.UnroutableError:
            print(f"Message unroutable: {order_data['id']}")
            return False
```

### Step 3.2: Consumer Implementation

```python
import json
import traceback

class MessageConsumer:
    def __init__(self, connection_params, queue_name, prefetch_count=10):
        self.connection = pika.BlockingConnection(connection_params)
        self.channel = self.connection.channel()
        self.channel.basic_qos(prefetch_count=prefetch_count)
        self.queue_name = queue_name

    def start_consuming(self):
        self.channel.basic_consume(
            queue=self.queue_name,
            on_message_callback=self.process_message,
            auto_ack=False  # Manual acknowledgment
        )
        self.channel.start_consuming()

    def process_message(self, channel, method, properties, body):
        try:
            message = json.loads(body)
            # Process the message
            result = self.handle_order(message)

            if result:
                channel.basic_ack(delivery_tag=method.delivery_tag)
            else:
                # Requeue for retry
                channel.basic_nack(delivery_tag=method.delivery_tag, requeue=True)

        except Exception as e:
            print(f"Error processing message: {e}")
            traceback.print_exc()
            # Reject without requeue (goes to DLQ)
            channel.basic_nack(delivery_tag=method.delivery_tag, requeue=False)
```

---

## 4. ElastiCache Redis Setup

### Step 4.1: Create Redis Replication Group

```bash
# Create subnet group
aws elasticache create-cache-subnet-group \
  --cache-subnet-group-name production-redis-subnet \
  --cache-subnet-group-description "Production Redis subnets" \
  --subnet-ids subnet-private-az1 subnet-private-az2 subnet-private-az3

# Create Redis replication group (Cluster Mode Disabled)
aws elasticache create-replication-group \
  --replication-group-id production-redis \
  --replication-group-description "Production Redis cluster" \
  --engine redis \
  --engine-version 7.1 \
  --cache-node-type cache.r6g.xlarge \
  --num-cache-clusters 3 \
  --cache-subnet-group-name production-redis-subnet \
  --security-group-ids sg-redis123456 \
  --automatic-failover-enabled \
  --multi-az-enabled \
  --at-rest-encryption-enabled \
  --transit-encryption-enabled \
  --auth-token "RedisAuthT0ken!Secure" \
  --snapshot-retention-limit 7 \
  --snapshot-window "03:00-05:00" \
  --preferred-maintenance-window "sun:05:00-sun:07:00" \
  --auto-minor-version-upgrade \
  --tags Key=Environment,Value=production Key=Team,Value=platform
```

### Step 4.2: Security Group for Redis

```bash
aws ec2 create-security-group \
  --group-name production-redis-sg \
  --description "Redis Production - Allow from app layer only" \
  --vpc-id vpc-0123456789abcdef0

# Allow Redis port from application
aws ec2 authorize-security-group-ingress \
  --group-id sg-redis123456 \
  --protocol tcp --port 6379 \
  --source-group sg-app123456
```

### Step 4.3: Get Connection Endpoints

```bash
aws elasticache describe-replication-groups \
  --replication-group-id production-redis \
  --query 'ReplicationGroups[0].{
    PrimaryEndpoint: NodeGroups[0].PrimaryEndpoint,
    ReaderEndpoint: NodeGroups[0].ReaderEndpoint,
    Status: Status
  }'

# Output:
# PrimaryEndpoint: production-redis.xxxxx.ng.0001.use1.cache.amazonaws.com:6379
# ReaderEndpoint: production-redis-ro.xxxxx.ng.0001.use1.cache.amazonaws.com:6379
```

---

## 5. Redis Cluster Mode Configuration

### Step 5.1: Create Cluster Mode Enabled (Sharded)

```bash
aws elasticache create-replication-group \
  --replication-group-id production-redis-cluster \
  --replication-group-description "Production Redis - Cluster Mode Enabled" \
  --engine redis \
  --engine-version 7.1 \
  --cache-node-type cache.r6g.xlarge \
  --num-node-groups 4 \
  --replicas-per-node-group 2 \
  --cache-subnet-group-name production-redis-subnet \
  --security-group-ids sg-redis123456 \
  --automatic-failover-enabled \
  --multi-az-enabled \
  --at-rest-encryption-enabled \
  --transit-encryption-enabled \
  --auth-token "RedisClusterAuth!Secure" \
  --snapshot-retention-limit 7 \
  --tags Key=Environment,Value=production

# Configuration endpoint (for cluster mode)
# production-redis-cluster.xxxxx.clustercfg.use1.cache.amazonaws.com:6379
```

### Step 5.2: Redis Connection (Python)

```python
import redis
from redis.cluster import RedisCluster

# Single node / Cluster Mode Disabled
redis_client = redis.Redis(
    host='production-redis.xxxxx.ng.0001.use1.cache.amazonaws.com',
    port=6379,
    password='RedisAuthT0ken!Secure',
    ssl=True,
    ssl_cert_reqs='required',
    decode_responses=True,
    socket_timeout=5,
    socket_connect_timeout=5,
    retry_on_timeout=True,
    health_check_interval=30
)

# Cluster Mode Enabled
redis_cluster = RedisCluster(
    host='production-redis-cluster.xxxxx.clustercfg.use1.cache.amazonaws.com',
    port=6379,
    password='RedisClusterAuth!Secure',
    ssl=True,
    decode_responses=True,
    skip_full_coverage_check=True
)
```

---

## 6. Caching Patterns & Implementation

### Step 6.1: Cache-Aside Pattern

```python
import json
from functools import wraps

def cache_aside(ttl=300, prefix=""):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            cache_key = f"{prefix}:{func.__name__}:{hash(str(args) + str(kwargs))}"

            # Try cache first
            cached = redis_client.get(cache_key)
            if cached:
                return json.loads(cached)

            # Cache miss - fetch from source
            result = func(*args, **kwargs)

            # Store in cache
            redis_client.setex(cache_key, ttl, json.dumps(result))
            return result
        return wrapper
    return decorator

# Usage
@cache_aside(ttl=600, prefix="orders")
def get_order_details(order_id):
    return db.query(f"SELECT * FROM orders WHERE id = {order_id}")
```

### Step 6.2: Rate Limiting with Redis

```python
def is_rate_limited(user_id, limit=100, window=60):
    """Sliding window rate limiter."""
    import time
    key = f"ratelimit:{user_id}"
    now = time.time()

    pipe = redis_client.pipeline()
    pipe.zremrangebyscore(key, 0, now - window)
    pipe.zadd(key, {f"{now}:{id(now)}": now})
    pipe.zcard(key)
    pipe.expire(key, window)
    results = pipe.execute()

    current_count = results[2]
    return current_count > limit
```

### Step 6.3: Session Storage

```python
def store_session(session_id, user_data, ttl=3600):
    key = f"session:{session_id}"
    redis_client.hset(key, mapping={
        'user_id': user_data['id'],
        'email': user_data['email'],
        'role': user_data['role'],
        'login_time': str(datetime.utcnow())
    })
    redis_client.expire(key, ttl)

def get_session(session_id):
    key = f"session:{session_id}"
    data = redis_client.hgetall(key)
    if data:
        redis_client.expire(key, 3600)  # Extend TTL on access
    return data
```

---

## 7. Monitoring & Alarms

### Step 7.1: Redis Alarms

```bash
# High Memory Usage
aws cloudwatch put-metric-alarm \
  --alarm-name "Redis-HighMemory" \
  --metric-name DatabaseMemoryUsagePercentage \
  --namespace AWS/ElastiCache \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=ReplicationGroupId,Value=production-redis \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-warnings

# High CPU
aws cloudwatch put-metric-alarm \
  --alarm-name "Redis-HighCPU" \
  --metric-name EngineCPUUtilization \
  --namespace AWS/ElastiCache \
  --statistic Average \
  --period 300 \
  --threshold 75 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --dimensions Name=ReplicationGroupId,Value=production-redis \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-warnings

# Evictions (indicates memory pressure)
aws cloudwatch put-metric-alarm \
  --alarm-name "Redis-Evictions" \
  --metric-name Evictions \
  --namespace AWS/ElastiCache \
  --statistic Sum \
  --period 300 \
  --threshold 100 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --dimensions Name=ReplicationGroupId,Value=production-redis \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-critical

# Replication Lag
aws cloudwatch put-metric-alarm \
  --alarm-name "Redis-ReplicationLag" \
  --metric-name ReplicationLag \
  --namespace AWS/ElastiCache \
  --statistic Maximum \
  --period 60 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --dimensions Name=ReplicationGroupId,Value=production-redis \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-warnings
```

### Step 7.2: RabbitMQ Alarms

```bash
# Queue Depth (messages backing up)
aws cloudwatch put-metric-alarm \
  --alarm-name "RabbitMQ-HighQueueDepth" \
  --metric-name MessageCount \
  --namespace AWS/AmazonMQ \
  --statistic Maximum \
  --period 60 \
  --threshold 10000 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --dimensions Name=Broker,Value=production-rabbitmq Name=Queue,Value=order-processing \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-critical

# Consumer Count (no consumers = messages pile up)
aws cloudwatch put-metric-alarm \
  --alarm-name "RabbitMQ-NoConsumers" \
  --metric-name ConsumerCount \
  --namespace AWS/AmazonMQ \
  --statistic Minimum \
  --period 60 \
  --threshold 1 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=Broker,Value=production-rabbitmq Name=Queue,Value=order-processing \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-critical
```

---

## 8. Scaling & High Availability

### Step 8.1: Scale Redis (Add Replicas)

```bash
# Add replica to existing replication group
aws elasticache increase-replica-count \
  --replication-group-id production-redis \
  --new-replica-count 3 \
  --apply-immediately

# Scale up node type
aws elasticache modify-replication-group \
  --replication-group-id production-redis \
  --cache-node-type cache.r6g.2xlarge \
  --apply-immediately
```

### Step 8.2: Scale Redis Cluster (Add Shards)

```bash
# Add shards (online resharding)
aws elasticache modify-replication-group-shard-configuration \
  --replication-group-id production-redis-cluster \
  --node-group-count 6 \
  --apply-immediately
```

---

## 9. Backup & Recovery

### Step 9.1: Redis Backup

```bash
# Create manual snapshot
aws elasticache create-snapshot \
  --replication-group-id production-redis \
  --snapshot-name "production-redis-backup-$(date +%Y%m%d)"

# Restore from snapshot
aws elasticache create-replication-group \
  --replication-group-id production-redis-restored \
  --replication-group-description "Restored from backup" \
  --snapshot-name "production-redis-backup-20240115" \
  --cache-node-type cache.r6g.xlarge \
  --engine redis
```

---

## 10. Production Best Practices

```bash
# Redis CLI commands for diagnostics
redis-cli -h endpoint -p 6379 --tls -a password INFO memory
redis-cli -h endpoint -p 6379 --tls -a password INFO clients
redis-cli -h endpoint -p 6379 --tls -a password --bigkeys
redis-cli -h endpoint -p 6379 --tls -a password SLOWLOG GET 10
redis-cli -h endpoint -p 6379 --tls -a password CLIENT LIST

# Check RabbitMQ management API
curl -u admin:password https://broker-endpoint/api/overview
curl -u admin:password https://broker-endpoint/api/queues
curl -u admin:password https://broker-endpoint/api/connections
```

---
