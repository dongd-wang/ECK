# Elastic Cloud on Kubernetes (ECK) on local


## Install Docker desktop

Install Docker Desktop from [Docker Desktop](https://www.docker.com/products/docker-desktop) and assign resource as follow
![](assets/Screenshot%202023-02-15%20at%2014.27.26.png)

## Install Minikube 
```
// M1 Mac
minikube start --driver=docker --memory='8192' --cpus='6' --container-runtime=containerd

// Inter Chip
minikube start --driver=docker --memory='8192' --cpus='6'

```

## Install elastic operator

```
kubectl create -f https://download.elastic.co/downloads/eck/2.6.1/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.6.1/operator.yaml
```


## Create Persistent Volumes

```
kubectl apply -f pv.yaml
```

```
#pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: eck-pv
  labels:
    app: eck
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /data
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - minikube
```

## Create Elasticsearch
```
kubectl apply -f elasticsearch.yaml
```

```
#elasticsearch.yaml
# This sample sets up an Elasticsearch cluster with 3 nodes.
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  # uncomment the lines below to copy the specified node labels as pod annotations and use it as an environment variable in the Pods
  #annotations:
  #  eck.k8s.elastic.co/downward-node-labels: "topology.kubernetes.io/zone"
  name: es1
spec:
  version: 8.6.1
  nodeSets:
  - name: default
    config:
      # most Elasticsearch configuration parameters are possible to set, e.g: node.attr.attr_name: attr_value
      node.roles: ["master", "data", "ingest", "ml"]
      # this allows ES to run on nodes even if their vm.max_map_count has not been increased, at a performance cost
      node.store.allow_mmap: false
      # uncomment the lines below to use the zone attribute from the node labels
      #cluster.routing.allocation.awareness.attributes: k8s_node_name,zone
      #node.attr.zone: ${ZONE}
    podTemplate:
      metadata:
        labels:
          # additional labels for pods
          app: eck
      spec:
        nodeName: minikube
        # this changes the kernel setting on the node to allow ES to use mmap
        # if you uncomment this init container you will likely also want to remove the
        # "node.store.allow_mmap: false" setting above
        initContainers:
        - name: sysctl
          securityContext:
            privileged: true
            runAsUser: 0
          command: ['sh', '-c', 'sysctl -w vm.max_map_count=262144']
        ###
        # uncomment the line below if you are using a service mesh such as linkerd2 that uses service account tokens for pod identification.
        # automountServiceAccountToken: true
        containers:
        - name: elasticsearch
          # specify resource limits and requests
          resources:
            limits:
              memory: 2Gi
              cpu: 2
          env:
          # uncomment the lines below to make the topology.kubernetes.io/zone annotation available as an environment variable and
          # use it as a cluster routing allocation attribute.
          #- name: ZONE
          #  valueFrom:
          #    fieldRef:
          #      fieldPath: metadata.annotations['topology.kubernetes.io/zone']
          - name: ES_JAVA_OPTS
            value: "-Xms1g -Xmx1g"
        #topologySpreadConstraints:
        #  - maxSkew: 1
        #    topologyKey: topology.kubernetes.io/zone
        #    whenUnsatisfiable: DoNotSchedule
        #    labelSelector:
        #      matchLabels:
        #        elasticsearch.k8s.elastic.co/cluster-name: elasticsearch-sample
        #        elasticsearch.k8s.elastic.co/statefulset-name: elasticsearch-sample-es-default
    count: 1
  #   # request 2Gi of persistent data storage for pods in this topology element
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data # Do not change this name unless you set up a volume mount for the data path.
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
        storageClassName: local-storage
  # # inject secure settings into Elasticsearch nodes from k8s secrets references
  # secureSettings:
  # - secretName: ref-to-secret
  # - secretName: another-ref-to-secret
  #   # expose only a subset of the secret keys (optional)
  #   entries:
  #   - key: value1
  #     path: newkey # project a key to a specific path (optional)
  http:
    service:
      spec:
        # expose this cluster Service with a LoadBalancer
        type: LoadBalancer
    tls:
      selfSignedCertificate:
        disabled: true
  #       # add a list of SANs into the self-signed HTTP certificate
  #       subjectAltNames:
  #       - ip: 192.168.1.2
  #       - ip: 192.168.1.3
  #       - dns: elasticsearch-sample.example.com
  #     certificate:
  #       # provide your own certificate
  #       secretName: my-cert
        
```

## Setup Kibana

```
kubectl apply -f kibana.yaml
```

```
# kibana.yaml
# This sample sets up an Elasticsearch cluster with 3 nodes.
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  # uncomment the lines below to copy the specified node labels as pod annotations and use it as an environment variable in the Pods
  #annotations:
  #  eck.k8s.elastic.co/downward-node-labels: "topology.kubernetes.io/zone"
  name: es1
spec:
  version: 8.6.1
  count: 1
  elasticsearchRef:
    name: es1
  http:
    service:
      spec:
        # expose this cluster Service with a LoadBalancer
        type: LoadBalancer
    # tls:
    #   selfSignedCertificate:
    #     disabled: true
```





