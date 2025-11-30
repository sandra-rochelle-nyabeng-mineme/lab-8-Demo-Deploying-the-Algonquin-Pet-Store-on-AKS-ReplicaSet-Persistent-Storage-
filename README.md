# lab-8-Demo-Deploying-the-Algonquin-Pet-Store-on-AKS-ReplicaSet-Persistent-Storage-

#### sandra rochelle nyabeng mineme
##  **YouTube Link:** https://youtu.be/V7M5TXJsXqs



# Overview

This lab demonstrates the deployment and improvement of the **Algonquin Pet Store (On Steroids)** microservices application on **Azure Kubernetes Service (AKS)**.

Two major tasks were completed:

- **Task 1:** Deploy all microservices using Docker images and Kubernetes manifests.  
- **Task 2:** Improve reliability by enabling **high availability** and **persistent storage** for MongoDB and RabbitMQ.


# Task 2 – Written Explanation of My Solution

## 1. MongoDB High Availability and Persistence

Originally, MongoDB was deployed as a single pod with no persistent storage. This meant the database was not fault-tolerant, and all data would be lost whenever the pod restarted or moved to another node.

To fix this, I redeployed MongoDB using a **StatefulSet** with **three replicas**. Each replica receives its own **PersistentVolumeClaim**, which ensures that its data is stored on durable disk storage instead of inside the container. The StatefulSet also provides stable hostnames for each pod, allowing MongoDB to form a proper ReplicaSet with one primary node and two secondary nodes.

These changes provide:
- **High availability**: the database keeps working even if a pod fails.
- **Data durability**: order data is preserved through restarts.
- **A more production-ready design** suitable for real-world applications.


## 2. RabbitMQ Persistent Storage

RabbitMQ was also originally deployed without persistence, meaning queued messages would be lost whenever the pod restarted. This broke the reliability of the order-processing workflow.

To resolve this, RabbitMQ was deployed as a **StatefulSet** with a **PersistentVolumeClaim** that stores its internal message data on disk. Even if RabbitMQ restarts, its queues and messages remain intact.

These improvements provide:
- **Durable message storage**
- **Reliable queue processing**
- **Consistent behavior even across pod failures**


## 3. End-to-End Reliability

After implementing persistence and high availability:
- Virtual Customer continuously generates new orders.
- Order Service sends messages to RabbitMQ reliably.
- Virtual Worker consumes and processes messages without losing them.
- MongoDB permanently stores all order documents.
- Store-admin displays orders in real time.

The system now behaves like a fully resilient cloud-native microservices architecture.


## 4. Azure Managed Services That Could Replace Self-Hosted MongoDB and RabbitMQ

In a real production environment, these backing services would typically be replaced with managed Azure alternatives to reduce operational overhead and improve reliability.

### Azure Cosmos DB (MongoDB API) — Replacement for MongoDB
**Purpose:**  
A fully managed NoSQL database service that is compatible with MongoDB applications.

**Why it’s a good fit:**  
- Built-in high availability across regions  
- Automatic backups and point-in-time restore  
- No need to manage ReplicaSets or storage manually  
- Easily scalable throughput and storage  
- Stronger reliability guarantees than self-hosted MongoDB

Cosmos DB removes all database maintenance effort while providing higher performance and resilience.


### Azure Service Bus — Replacement for RabbitMQ
**Purpose:**  
A fully managed enterprise messaging service used for queuing and pub/sub communication.

**Why it’s a good fit:**  
- Fully durable and fault-tolerant messaging  
- Automatic scaling with high throughput  
- Built-in dead-letter queues and message retries  
- No need to manage StatefulSets, volumes, or plugins  
- Designed for large distributed applications

Service Bus provides robust and reliable messaging without the operational complexity of maintaining RabbitMQ.


## Conclusion

Task 2 improved the reliability of the Algonquin Pet Store by adding persistence and high availability to MongoDB and RabbitMQ.  
Additionally, Azure Cosmos DB and Azure Service Bus offer fully managed alternatives that provide greater scalability, durability, and operational simplicity for real-world cloud deployments.


# yaml file

## task 1

```
# StatefulSet for MongoDB
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: mongodb # Associated headless service for DNS resolution
  replicas: 1 # Number of MongoDB replicas
  selector:
    matchLabels:
      app: mongodb # Labels to match for this StatefulSet
  template:
    metadata:
      labels:
        app: mongodb # Pod label for identification
    spec:
      nodeSelector:
        "kubernetes.io/os": linux # Run only on Linux nodes
      containers:
        - name: mongodb
          image: mongo:4.2 # MongoDB container image version
          ports:
            - containerPort: 27017 # Default MongoDB port
              name: mongodb
          resources: # Resource requests and limits for MongoDB
            requests:
              cpu: 5m # Minimum CPU
              memory: 75Mi # Minimum Memory
            limits:
              cpu: 25m # Maximum CPU
              memory: 1024Mi # Maximum Memory
          livenessProbe: # Probe to check MongoDB health
            exec:
              command:
                - mongosh
                - --eval
                - db.runCommand('ping').ok
            initialDelaySeconds: 5 # Wait time before starting probe
            periodSeconds: 5 # Interval between probes
---
# Service to expose MongoDB
apiVersion: v1
kind: Service
metadata:
  name: mongodb
spec:
  ports:
    - port: 27017 # MongoDB service port
  selector:
    app: mongodb # Target pods with label `app: mongodb`
  type: ClusterIP # Internal cluster service
---
# StatefulSet for RabbitMQ
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
spec:
  serviceName: rabbitmq # Headless service for RabbitMQ
  replicas: 1 # Single replica
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: rabbitmq
          image: rabbitmq:3-management # RabbitMQ image with management UI
          ports:
            - containerPort: 5672 # AMQP protocol port
              name: rabbitmq-amqp
            - containerPort: 15672 # HTTP management UI port
              name: rabbitmq-http
          env: # Environment variables for RabbitMQ
            - name: RABBITMQ_DEFAULT_USER
              value: "username" # Default username
            - name: RABBITMQ_DEFAULT_PASS
              value: "password" # Default password
          resources: # Resource limits and requests
            requests:
              cpu: 10m
              memory: 128Mi
            limits:
              cpu: 250m
              memory: 256Mi
          volumeMounts: # Mount configuration for RabbitMQ plugins
            - name: rabbitmq-enabled-plugins
              mountPath: /etc/rabbitmq/enabled_plugins
              subPath: enabled_plugins
      volumes:
        - name: rabbitmq-enabled-plugins
          configMap:
            name: rabbitmq-enabled-plugins
            items:
              - key: rabbitmq_enabled_plugins
                path: enabled_plugins
---
# Service for RabbitMQ
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq
spec:
  selector:
    app: rabbitmq
  ports:
    - name: rabbitmq-amqp
      port: 5672
      targetPort: 5672
    - name: rabbitmq-http
      port: 15672
      targetPort: 15672
  type: ClusterIP
---
# Deployment for Order Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 1 # Single replica
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: order-service
          image: laroche237/order-service-l8:v1 # Custom image for order service
          ports:
            - containerPort: 3000 # Order service listens on this port
          env: # Environment variables for configuration
            - name: ORDER_QUEUE_HOSTNAME
              value: "rabbitmq"
            - name: ORDER_QUEUE_PORT
              value: "5672"
            - name: ORDER_QUEUE_USERNAME
              value: "username"
            - name: ORDER_QUEUE_PASSWORD
              value: "password"
            - name: ORDER_QUEUE_NAME
              value: "orders"
            - name: FASTIFY_ADDRESS
              value: "0.0.0.0"
          resources: # Resource allocation
            requests:
              cpu: 1m
              memory: 50Mi
            limits:
              cpu: 100m
              memory: 256Mi
          startupProbe: # Initial health check
            httpGet:
              path: /health
              port: 3000
            failureThreshold: 5
            initialDelaySeconds: 20
            periodSeconds: 10
          readinessProbe: # Ready status probe
            httpGet:
              path: /health
              port: 3000
            failureThreshold: 3
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe: # Ongoing health check
            httpGet:
              path: /health
              port: 3000
            failureThreshold: 5
            initialDelaySeconds: 3
            periodSeconds: 3
      initContainers: # Initialization container to wait for RabbitMQ
        - name: wait-for-rabbitmq
          image: busybox
          command:
            [
              "sh",
              "-c",
              "until nc -zv rabbitmq 5672; do echo waiting for rabbitmq; sleep 2; done;",
            ]
          resources:
            requests:
              cpu: 1m
              memory: 50Mi
            limits:
              cpu: 100m
              memory: 256Mi
---
# Service for Order Service
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 3000
      targetPort: 3000
  selector:
    app: order-service
---
# Deployment for Makeline Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: makeline-service
spec:
  replicas: 1 # Single replica
  selector:
    matchLabels:
      app: makeline-service
  template:
    metadata:
      labels:
        app: makeline-service
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: makeline-service
          image: laroche237/makeline-service-l8:v1 # Custom Makeline service image
          ports:
            - containerPort: 3001 # Makeline service listens on this port
          env: # Environment variables for configuration
            - name: ORDER_QUEUE_URI
              value: "amqp://rabbitmq:5672" # RabbitMQ connection string
            - name: ORDER_QUEUE_USERNAME
              value: "username"
            - name: ORDER_QUEUE_PASSWORD
              value: "password"
            - name: ORDER_QUEUE_NAME
              value: "orders"
            - name: ORDER_DB_URI
              value: "mongodb://mongodb:27017" # MongoDB connection string
            - name: ORDER_DB_NAME
              value: "orderdb" # Database name
            - name: ORDER_DB_COLLECTION_NAME
              value: "orders" # Collection name
          resources: # Resource requests and limits
            requests:
              cpu: 1m
              memory: 6Mi
            limits:
              cpu: 5m
              memory: 20Mi
          startupProbe: # Initial health check
            httpGet:
              path: /health
              port: 3001
            failureThreshold: 10
            periodSeconds: 5
          readinessProbe: # Ready status probe
            httpGet:
              path: /health
              port: 3001
            failureThreshold: 3
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe: # Ongoing health check
            httpGet:
              path: /health
              port: 3001
            failureThreshold: 5
            initialDelaySeconds: 3
            periodSeconds: 3
---
# Service for Makeline Service
apiVersion: v1
kind: Service
metadata:
  name: makeline-service
spec:
  type: ClusterIP # Internal cluster service
  ports:
    - name: http
      port: 3001 # Service port
      targetPort: 3001
  selector:
    app: makeline-service
---
# Deployment for Product Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
spec:
  replicas: 1 # Single replica
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: product-service
          image: laroche237/product-service-l8:v1 # Custom product service image
          ports:
            - containerPort: 3002 # Product service listens on this port
          env: # Environment variables for configuration
            - name: AI_SERVICE_URL
              value: "http://ai-service:5001/" # URL for AI service integration
          resources: # Resource requests and limits
            requests:
              cpu: 1m
              memory: 1Mi
            limits:
              cpu: 2m
              memory: 20Mi
          readinessProbe: # Ready status probe
            httpGet:
              path: /health
              port: 3002
            failureThreshold: 3
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe: # Ongoing health check
            httpGet:
              path: /health
              port: 3002
            failureThreshold: 5
            initialDelaySeconds: 3
            periodSeconds: 3
---
# Service for Product Service
apiVersion: v1
kind: Service
metadata:
  name: product-service
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 3002 # Service port
      targetPort: 3002
  selector:
    app: product-service
---
# Deployment for Store Front
apiVersion: apps/v1
kind: Deployment
metadata:
  name: store-front
spec:
  replicas: 1
  selector:
    matchLabels:
      app: store-front
  template:
    metadata:
      labels:
        app: store-front
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: store-front
          image: laroche237/store-front-l8:v1 # Front-end store application
          ports:
            - containerPort: 8080 # Front-end app port
              name: store-front
          env: # Environment variables for backend URLs
            - name: VUE_APP_ORDER_SERVICE_URL
              value: "http://order-service:3000/"
            - name: VUE_APP_PRODUCT_SERVICE_URL
              value: "http://product-service:3002/"
          resources: # Resource requests and limits
            requests:
              cpu: 1m
              memory: 200Mi
            limits:
              cpu: 1000m
              memory: 512Mi
          startupProbe: # Initial health check
            httpGet:
              path: /health
              port: 8080
            failureThreshold: 3
            initialDelaySeconds: 5
            periodSeconds: 5
          readinessProbe: # Ready status probe
            httpGet:
              path: /health
              port: 8080
            failureThreshold: 3
            initialDelaySeconds: 3
            periodSeconds: 3
          livenessProbe: # Ongoing health check
            httpGet:
              path: /health
              port: 8080
            failureThreshold: 5
            initialDelaySeconds: 3
            periodSeconds: 3
---
# Service for Store Front
apiVersion: v1
kind: Service
metadata:
  name: store-front
spec:
  ports:
    - port: 80 # Service port exposed outside the cluster
      targetPort: 8080 # Target container port
  selector:
    app: store-front
  type: LoadBalancer # Expose service externally via load balancer
---
# Deployment for Store Admin
apiVersion: apps/v1
kind: Deployment
metadata:
  name: store-admin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: store-admin
  template:
    metadata:
      labels:
        app: store-admin
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: store-admin
          image: laroche237/store-admin-l8:v1 # Admin interface for store
          ports:
            - containerPort: 8081 # Admin app port
              name: store-admin
          env: # Environment variables for backend URLs
            - name: VUE_APP_PRODUCT_SERVICE_URL
              value: "http://product-service:3002/"
            - name: VUE_APP_MAKELINE_SERVICE_URL
              value: "http://makeline-service:3001/"
          resources: # Resource requests and limits
            requests:
              cpu: 1m
              memory: 200Mi
            limits:
              cpu: 1000m
              memory: 512Mi
          startupProbe: # Initial health check
            httpGet:
              path: /health
              port: 8081
            failureThreshold: 3
            initialDelaySeconds: 5
            periodSeconds: 5
          readinessProbe: # Ready status probe
            httpGet:
              path: /health
              port: 8081
            failureThreshold: 3
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe: # Ongoing health check
            httpGet:
              path: /health
              port: 8081
            failureThreshold: 5
            initialDelaySeconds: 3
            periodSeconds: 3
---
# Service for Store Admin
apiVersion: v1
kind: Service
metadata:
  name: store-admin
spec:
  ports:
    - port: 80 # Service port exposed outside the cluster
      targetPort: 8081
  selector:
    app: store-admin
  type: LoadBalancer # Expose service externally via load balancer
---
# Deployment for AI Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ai-service
  template:
    metadata:
      labels:
        app: ai-service
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: ai-service
          image: laroche237/ai-service-l8:v1
          ports:
            - containerPort: 5001
          env:
            - name: USE_AZURE_OPENAI # set to True for Azure OpenAI, False for Public OpenAI
              value: "True"
            - name: AZURE_OPENAI_API_VERSION
              value: "2024-07-01-preview"
            - name: AZURE_OPENAI_GPT_DEPLOYMENT_NAME # required if using Azure OpenAI
              value: ""
            - name: AZURE_OPENAI_GPT_ENDPOINT # required if using Azure OpenAI
              value: ""
            - name: AZURE_OPENAI_DALLE_ENDPOINT
              value: ""
            - name: AZURE_OPENAI_DALLE_DEPLOYMENT_NAME
              value: ""
            - name: OPENAI_GPT_API_KEY # always required
              valueFrom:
                secretKeyRef:
                  name: openai-gpt-api-secret
                  key: OPENAI_GPT_API_KEY
            - name: OPENAI_DALLE_API_KEY # always required
              valueFrom:
                secretKeyRef:
                  name: openai-dalle-api-secret
                  key: OPENAI_DALLE_API_KEY
          resources:
            requests:
              cpu: 20m
              memory: 50Mi
            limits:
              cpu: 50m
              memory: 128Mi
          startupProbe:
            httpGet:
              path: /health
              port: 5001
            initialDelaySeconds: 60
            failureThreshold: 3
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /health
              port: 5001
            initialDelaySeconds: 3
            failureThreshold: 10
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 5001
            initialDelaySeconds: 3
            failureThreshold: 10
            periodSeconds: 10
---
# Service for AI Service
apiVersion: v1
kind: Service
metadata:
  name: ai-service
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 5001
      targetPort: 5001
  selector:
    app: ai-service

```


## task 2

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: rabbitmq-enabled-plugins
data:
  enabled_plugins: |
    [rabbitmq_management,rabbitmq_prometheus,rabbitmq_amqp1_0].

---
# -----------------------------------------------------
# HEADLESS SERVICE for MongoDB
# -----------------------------------------------------
apiVersion: v1
kind: Service
metadata:
  name: mongodb
spec:
  clusterIP: None
  selector:
    app: mongodb
  ports:
    - port: 27017
      name: mongodb
---
# -----------------------------------------------------
# STATEFULSET for MongoDB (Replica Set + PVC)
# -----------------------------------------------------
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: mongodb
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: mongo:6
          args: ["--replSet=rs0", "--bind_ip_all"]
          ports:
            - containerPort: 27017
              name: mongodb
          volumeMounts:
            - name: mongo-data
              mountPath: /data/db
  volumeClaimTemplates:
    - metadata:
        name: mongo-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources: 
          requests:
            storage: 5Gi

---
# -----------------------------------------------------
# SERVICE for RabbitMQ
# -----------------------------------------------------
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq
spec:
  selector:
    app: rabbitmq
  ports:
    - port: 5672
      name: amqp
    - port: 15672
      name: http
---
# -----------------------------------------------------
# STATEFULSET for RabbitMQ (PVC persistence)
# -----------------------------------------------------
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
spec:
  serviceName: rabbitmq
  replicas: 1
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      containers:
        - name: rabbitmq
          image: rabbitmq:3-management
          ports:
            - containerPort: 5672
              name: amqp
            - containerPort: 15672
              name: http
          volumeMounts:
            - name: rabbitmq-data
              mountPath: /var/lib/rabbitmq
            - name: rabbitmq-enabled-plugins
              mountPath: /etc/rabbitmq/enabled_plugins
              subPath: enabled_plugins
      volumes:
        - name: rabbitmq-enabled-plugins
          configMap:
            name: rabbitmq-enabled-plugins
  volumeClaimTemplates:
    - metadata:
        name: rabbitmq-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 5Gi


---
# Deployment for Order Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 1 # Single replica
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: order-service
          image: laroche237/order-service-l8:v1 # Custom image for order service
          ports:
            - containerPort: 3000 # Order service listens on this port
          env: # Environment variables for configuration
            - name: ORDER_QUEUE_HOSTNAME
              value: "rabbitmq"
            - name: ORDER_QUEUE_PORT
              value: "5672"
            - name: ORDER_QUEUE_USERNAME
              value: "guest"
            - name: ORDER_QUEUE_PASSWORD
              value: "guest"
            - name: ORDER_QUEUE_NAME
              value: "orders"
            - name: FASTIFY_ADDRESS
              value: "0.0.0.0"
          resources: # Resource allocation
            requests:
              cpu: 1m
              memory: 50Mi
            limits:
              cpu: 100m
              memory: 256Mi
          startupProbe: # Initial health check
            httpGet:
              path: /health
              port: 3000
            failureThreshold: 5
            initialDelaySeconds: 20
            periodSeconds: 10
          readinessProbe: # Ready status probe
            httpGet:
              path: /health
              port: 3000
            failureThreshold: 3
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe: # Ongoing health check
            httpGet:
              path: /health
              port: 3000
            failureThreshold: 5
            initialDelaySeconds: 3
            periodSeconds: 3
      initContainers: # Initialization container to wait for RabbitMQ
        - name: wait-for-rabbitmq
          image: busybox
          command:
            [
              "sh",
              "-c",
              "until nc -zv rabbitmq 5672; do echo waiting for rabbitmq; sleep 2; done;",
            ]
          resources:
            requests:
              cpu: 1m
              memory: 50Mi
            limits:
              cpu: 100m
              memory: 256Mi
---
# Service for Order Service
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 3000
      targetPort: 3000
  selector:
    app: order-service
---
# Deployment for Makeline Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: makeline-service
spec:
  replicas: 1 # Single replica
  selector:
    matchLabels:
      app: makeline-service
  template:
    metadata:
      labels:
        app: makeline-service
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: makeline-service
          image: laroche237/makeline-service-l8:v1 # Custom Makeline service image
          ports:
            - containerPort: 3001 # Makeline service listens on this port
          env: # Environment variables for configuration
            - name: ORDER_QUEUE_URI
              value: "amqp://rabbitmq:5672" # RabbitMQ connection string
            - name: ORDER_QUEUE_USERNAME
              value: "guest"
            - name: ORDER_QUEUE_PASSWORD
              value: "guest"
            - name: ORDER_QUEUE_NAME
              value: "orders"
            - name: ORDER_DB_URI
              value: "mongodb://mongodb:27017" # MongoDB connection string
            - name: ORDER_DB_NAME
              value: "orderdb" # Database name
            - name: ORDER_DB_COLLECTION_NAME
              value: "orders" # Collection name
          resources: # Resource requests and limits
            requests:
              cpu: 1m
              memory: 6Mi
            limits:
              cpu: 5m
              memory: 20Mi
          startupProbe: # Initial health check
            httpGet:
              path: /health
              port: 3001
            failureThreshold: 10
            periodSeconds: 5
          readinessProbe: # Ready status probe
            httpGet:
              path: /health
              port: 3001
            failureThreshold: 3
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe: # Ongoing health check
            httpGet:
              path: /health
              port: 3001
            failureThreshold: 5
            initialDelaySeconds: 3
            periodSeconds: 3
---
# Service for Makeline Service
apiVersion: v1
kind: Service
metadata:
  name: makeline-service
spec:
  type: ClusterIP # Internal cluster service
  ports:
    - name: http
      port: 3001 # Service port
      targetPort: 3001
  selector:
    app: makeline-service
---
# Deployment for Product Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
spec:
  replicas: 1 # Single replica
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: product-service
          image: laroche237/product-service-l8:v1 # Custom product service image
          ports:
            - containerPort: 3002 # Product service listens on this port
          env: # Environment variables for configuration
            - name: AI_SERVICE_URL
              value: "http://ai-service:5001/" # URL for AI service integration
          resources: # Resource requests and limits
            requests:
              cpu: 1m
              memory: 1Mi
            limits:
              cpu: 2m
              memory: 20Mi
          readinessProbe: # Ready status probe
            httpGet:
              path: /health
              port: 3002
            failureThreshold: 3
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe: # Ongoing health check
            httpGet:
              path: /health
              port: 3002
            failureThreshold: 5
            initialDelaySeconds: 3
            periodSeconds: 3
---
# Service for Product Service
apiVersion: v1
kind: Service
metadata:
  name: product-service
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 3002 # Service port
      targetPort: 3002
  selector:
    app: product-service
---
# Deployment for Store Front
apiVersion: apps/v1
kind: Deployment
metadata:
  name: store-front
spec:
  replicas: 1
  selector:
    matchLabels:
      app: store-front
  template:
    metadata:
      labels:
        app: store-front
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: store-front
          image: laroche237/store-front-l8:v1 # Front-end store application
          ports:
            - containerPort: 8080 # Front-end app port
              name: store-front
          env: # Environment variables for backend URLs
            - name: VUE_APP_ORDER_SERVICE_URL
              value: "http://order-service:3000/"
            - name: VUE_APP_PRODUCT_SERVICE_URL
              value: "http://product-service:3002/"
          resources: # Resource requests and limits
            requests:
              cpu: 1m
              memory: 200Mi
            limits:
              cpu: 1000m
              memory: 512Mi
          startupProbe: # Initial health check
            httpGet:
              path: /health
              port: 8080
            failureThreshold: 3
            initialDelaySeconds: 5
            periodSeconds: 5
          readinessProbe: # Ready status probe
            httpGet:
              path: /health
              port: 8080
            failureThreshold: 3
            initialDelaySeconds: 3
            periodSeconds: 3
          livenessProbe: # Ongoing health check
            httpGet:
              path: /health
              port: 8080
            failureThreshold: 5
            initialDelaySeconds: 3
            periodSeconds: 3
---
# Service for Store Front
apiVersion: v1
kind: Service
metadata:
  name: store-front
spec:
  ports:
    - port: 80 # Service port exposed outside the cluster
      targetPort: 8080 # Target container port
  selector:
    app: store-front
  type: LoadBalancer # Expose service externally via load balancer
---
# Deployment for Store Admin
apiVersion: apps/v1
kind: Deployment
metadata:
  name: store-admin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: store-admin
  template:
    metadata:
      labels:
        app: store-admin
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: store-admin
          image: laroche237/store-admin-l8:v1 # Admin interface for store
          ports:
            - containerPort: 8081 # Admin app port
              name: store-admin
          env: # Environment variables for backend URLs
            - name: VUE_APP_PRODUCT_SERVICE_URL
              value: "http://product-service:3002/"
            - name: VUE_APP_MAKELINE_SERVICE_URL
              value: "http://makeline-service:3001/"
          resources: # Resource requests and limits
            requests:
              cpu: 1m
              memory: 200Mi
            limits:
              cpu: 1000m
              memory: 512Mi
          startupProbe: # Initial health check
            httpGet:
              path: /health
              port: 8081
            failureThreshold: 3
            initialDelaySeconds: 5
            periodSeconds: 5
          readinessProbe: # Ready status probe
            httpGet:
              path: /health
              port: 8081
            failureThreshold: 3
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe: # Ongoing health check
            httpGet:
              path: /health
              port: 8081
            failureThreshold: 5
            initialDelaySeconds: 3
            periodSeconds: 3
---
# Service for Store Admin
apiVersion: v1
kind: Service
metadata:
  name: store-admin
spec:
  ports:
    - port: 80 # Service port exposed outside the cluster
      targetPort: 8081
  selector:
    app: store-admin
  type: LoadBalancer # Expose service externally via load balancer
---
# Deployment for AI Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ai-service
  template:
    metadata:
      labels:
        app: ai-service
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: ai-service
          image: laroche237/ai-service-l8:v1
          ports:
            - containerPort: 5001
          env:
            - name: USE_AZURE_OPENAI # set to True for Azure OpenAI, False for Public OpenAI
              value: "True"
            - name: AZURE_OPENAI_API_VERSION
              value: "2024-07-01-preview"
            - name: AZURE_OPENAI_GPT_DEPLOYMENT_NAME # required if using Azure OpenAI
              value: ""
            - name: AZURE_OPENAI_GPT_ENDPOINT # required if using Azure OpenAI
              value: ""
            - name: AZURE_OPENAI_DALLE_ENDPOINT
              value: ""
            - name: AZURE_OPENAI_DALLE_DEPLOYMENT_NAME
              value: ""
            - name: OPENAI_GPT_API_KEY # always required
              valueFrom:
                secretKeyRef:
                  name: openai-gpt-api-secret
                  key: OPENAI_GPT_API_KEY
            - name: OPENAI_DALLE_API_KEY # always required
              valueFrom:
                secretKeyRef:
                  name: openai-dalle-api-secret
                  key: OPENAI_DALLE_API_KEY
          resources:
            requests:
              cpu: 20m
              memory: 50Mi
            limits:
              cpu: 50m
              memory: 128Mi
          startupProbe:
            httpGet:
              path: /health
              port: 5001
            initialDelaySeconds: 60
            failureThreshold: 3
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /health
              port: 5001
            initialDelaySeconds: 3
            failureThreshold: 10
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 5001
            initialDelaySeconds: 3
            failureThreshold: 10
            periodSeconds: 10
---
# Service for AI Service
apiVersion: v1
kind: Service
metadata:
  name: ai-service
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 5001
      targetPort: 5001
  selector:
    app: ai-service

```