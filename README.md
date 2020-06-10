# Table of Contents
- [Introduction](#introduction) 
- [Instructions](#instructions) 
  - [Step 1](#step-1)
  - [Step 2](#step-2)
  - [Step 3](#step-3)
  - [Step 4](#step-4)
  - [Step 5](#step-5)
  - [Step 5a](#step-5a)
  - [Step 5b](#step-5b)
  - [Step 5c](#step-5c)
  - [Step 6](#step-6)
  - [Step 7](#step-7)
  - [Step 8](#step-8)
  - [Step 9](#step-9)
  - [Step 10](#step-10)

# Introduction 
This folder consists of 11 yaml files. All files are numbered in proper order. All the files when executed in order will create an elk stack with logstash reading from azure event hub, elastic search processing the indexes and kibana displaying the results in gui.

This folder has following files.

- 0-create-ns.yaml -- This file creates the namespace in kubernetes cluster.

- 1-pv.yaml -- This file creates persistent volume for index/logs storage.

- 2-elastic-cfg-map.yaml -- This file creates configmap for elasticsearch.

- 3-elastic.service.yaml -- This file creates loadbalancer ip for elasticsearch.

- 4-elastic-stateful-set.yaml -- This file creates stateful set for elasticsearch.

- 5-kibana-configmap.yaml -- This file creates configmap for kibana.

- 6-kibana-service.yaml -- This file creates loadbalancer ip for kibana.

- 7-kibana-deployment.yaml -- This file creates deployment for kibana.

- 8-logstash-configmap.yaml -- This file creates configmap for logstash.

- 9-logstash-service.yaml -- This file creates loadbalancer ip for logstash.

- 10-logstash-deployment.yaml -- This file creates deployment for logstash.

# Instructions
## Step 1
- Create a namespace in kubernetes cluster. Name can be changed under metadata
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: elk-logging
  labels:
    name: development
```

## Step 2
- Create a persistent volume.
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: elasticsearch-data-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
   requests:
    storage: 5Gi
```

## Step 3
- Create a configmap for elasticsearch.
  - This configmap will have all the necessary settings to run elasticsearch.
  - For our lab we are using trial version which gives you access to all features for 30 days.
  - After the trial version expiration, we can move to basic free version.
  - We are only using 1 master node. We can have multiple nodes though.
```yaml
data:
  elasticsearch.yml: |
    cluster.name: "elasticsearch-cluster"
    network.host: 0.0.0.0
    discovery.zen.minimum_master_nodes: 1
    xpack.license.self_generated.type: trial
    node.max_local_storage_nodes: 1
    xpack.security.enabled: true
    xpack.monitoring.collection.enabled: true
```

## Step 4
- Create loadbalancer for the elasticsearch.
  - We are using default elasticsearch port for this service.
```yaml
ports:
  - name: http
    port: 9200
    targetPort: 9200
    protocol: TCP
  - name: transport
    port: 9300
    targetPort: 9300
    protocol: TCP
```
  - We need to make sure that selector has the correct name i.e. the name we will use for elastic search stateful set
```yaml
selector:
    service: elasticsearch
```

## Step 5
#### Step 5a
- Create stateful set for elasticsearch.
  - We are going to use the correct PersistentVolumeClaim that we created in the - [Step 2](#step-2)
```yaml
volumeClaimTemplates:
  - metadata:
      name: elasticsearch-data-claim
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: defaultku
      resources:
        requests:
          storage: 2Gi
```

#### Step 5b
- Now that we are done with elasticsearch, we can see that its working by visiting the http://{serviceip}:9200
- Once elasticsearch is verified, we need to set the passwords.
  - Open bash session on elasticsearch using the following command
  ```command 
  kubectl exec -ti elasticsearch-0 bash
  ```
  - Once bash is open, run the following command and set the password. For our lab we are using 'Password1$'. If a different password is used, please change it in the yaml files.
  ```command
  bin/elasticsearch-setup-passwords interactive
  ```
- Once passwords are set, we need to activate the trial version.
  - For this, I used postman, but you can use any agent that can make a post request.
  - Append /_license/start_trial?acknowledge=true at the end of elasticsearch url.
  - For credentials use the user of your choice, 'UserXYZ', and password of your choice, 'PasswordXYZ'.
- Elasticsearch setup is complete now.

#### Step 5c
- Create configmap for kibana 
- Make sure to add the following
  - Elasticsearch url
  - Username: UserXYZ
  - Password: PasswordXYZ
```yaml
data:
  kibana.yml: |
    server.name: kibana
    server.host: "0"
    elasticsearch.url: http://elasticsearch:9200
    xpack.monitoring.ui.container.elasticsearch.enabled: true
    elasticsearch.username: UserXYZ
    elasticsearch.password: PasswordXYZ
```

## Step 6
- Create Loadbalancer service for kibana
- Make sure to use the proper selector
```yaml
spec:
  type: LoadBalancer
  selector:
    component: kibana
  ports:
  - name: http
    port: 80
    targetPort: http
```

## Step 7
- Create Deployment for kibana
- Make sure to choose the correct names etc.
- Make sure to use correct configmap
```yaml
 spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:6.4.1
        resources:
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        ports:
        - containerPort: 5601
          name: http
        volumeMounts:
        - name: kibana-configmap
          mountPath: /usr/share/kibana/config/kibana.yml
          subPath: kibana.yml
        terminationMessagePath: "/dev/termination-log"
        terminationMessagePolicy: File
        imagePullPolicy: Always
        securityContext:
          privileged: false
      volumes:
      - name: kibana-configmap
        configMap:
            name: kibana-configmap
```

## Step 8
- Create configmap for logstash
- Make sure to specify the source and destination as input and output
- Make sure to specify the correct username and password for elastic search
- You can specify the name for the index as well
- filter is used to transform the data at the time of indexing
  - Following example as json filter which converts a json formatted string into json object
```yaml
data:

  logstash.yml: |
    xpack.monitoring.elasticsearch.url: http://elasticsearch:9200
    dead_letter_queue.enable: true
    xpack.monitoring.enabled: true
    xpack.monitoring.elasticsearch.username: UserXYZ
    xpack.monitoring.elasticsearch.password: PasswordXYZ
  pipelines.yml: |
    - pipeline.id: azureeventhubs
      path.config: "/usr/share/logstash/azureeventhubs.cfg"

  azureeventhubs.cfg: |
    input {
      azure_event_hubs {
        event_hub_connections => [Endpoint connection]
        threads => 2
        decorate_events => true
        consumer_group => "$Default"
        storage_connection => "Storage connection"
        storage_container => "eventstorage"
        }
    }
    filter {
      json {
        source => "message"
        target => "newmessage"
      }
    }
    output {
      elasticsearch {
        hosts => [ "elasticsearch:9200" ]
        user => "elastic"
        password => "*******"
        index => "azureeventhub-%{+YYYY.MM.dd}"
      }
    }
  logstash.conf: |
```

## Step 9
- Create the clusterip service for logstash
- Make sure to use the correct selector
```yaml
spec:
  type: ClusterIP
  selector:
    component: logstash
  ports:
  - name: http
    port: 80
    targetPort: http
```

## Step 10
- Create a deployment for logstash
- Make sure to use the correct configmaps
```yaml
spec:
  replicas: 1
  selector:
    matchLabels:
     component: logstash
  template:
    metadata:
      labels:
        component: logstash
    spec:
      containers:
      - name: logstash
        image: docker.elastic.co/logstash/logstash:6.4.1
        volumeMounts:
        - name: logstash-configmap
          mountPath: /usr/share/logstash/config/logstash.yml
          subPath: logstash.yml
        - name: logstash-configmap
          mountPath: /usr/share/logstash/pipeline/logstash.conf
          subPath: logstash.conf
        - name: logstash-configmap
          mountPath: /usr/share/logstash/azureeventhubs.cfg
          subPath: azureeventhubs.cfg
        - name: logstash-configmap
          mountPath: /usr/share/logstash/config/pipelines.yml
          subPath: pipelines.yml
        resources:
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        ports:
        - containerPort: 5601
          name: http
      volumes:
        - name: logstash-configmap
          configMap:
            name: logstash-configmap
```
