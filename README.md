# grpc-proxyless-traffic-management-01
testing out https://cloud.google.com/service-mesh/docs/gateway/proxyless-grpc-mesh

### set up environment 
> note: this assumes a cluster has been already created and added to a fleet 

```
export PROJECT_ID=grpc-proxyless-csm-01 # replace with your project
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
export CLUSTER_NAME=grpc-proxyless-autopilot-02 # replace with your cluster
export REGION=us-central1 # replace with your region

gcloud config set project ${PROJECT_ID}

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member "group:${PROJECT_ID}.svc.id.goog:/allAuthenticatedUsers/" \
    --role "roles/trafficdirector.client"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
--member "serviceAccount:service-${PROJECT_NUMBER}@container-engine-robot.iam.gserviceaccount.com" \
--role "roles/container.developer"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
--member "serviceAccount:service-${PROJECT_NUMBER}@container-engine-robot.iam.gserviceaccount.com" \
--role "roles/compute.networkAdmin"

gcloud container fleet mesh enable --project ${PROJECT_ID}

gcloud alpha container fleet mesh update \
--config-api gateway \
--memberships ${CLUSTER_NAME} \
--project ${PROJECT_ID}

# install gRPCroute
curl https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.1.0/config/crd/standard/gateway.networking.k8s.io_grpcroutes.yaml \
| kubectl apply -f -
```

# install gRPC service (Python in this case)

```
kubectl apply -f - <<EOF
---
apiVersion: v1
kind: Service
metadata:
  name: helloworld
spec:
  selector:
    app: psm-grpc-server
  ports:
  - port: 50051
    targetPort: 50051
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: psm-grpc-server
  labels:
    app: psm-grpc-server
spec:
  replicas: 4
  selector:
    matchLabels:
      app: psm-grpc-server
  template:
    metadata:
      labels:
        app: psm-grpc-server
    spec:
      containers:
      - name: psm-grpc-server
        image: grpc/csm-o11y-example-python-server:latest
        imagePullPolicy: Always
        args:
          - "--port=50051"
        ports:
          - containerPort: 50051
        env:
          - name: GRPC_XDS_BOOTSTRAP
            value: "/tmp/grpc-xds/td-grpc-bootstrap.json"
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: NAMESPACE_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: CSM_WORKLOAD_NAME
            value: psm-grpc-server
          - name: OTEL_RESOURCE_ATTRIBUTES
            value: k8s.pod.name=\$(POD_NAME),k8s.namespace.name=\$(NAMESPACE_NAME),k8s.container.name=\$(CONTAINER_NAME)
        volumeMounts:
          - mountPath: /tmp/grpc-xds/
            name: grpc-td-conf
            readOnly: true
      initContainers:
        - name: grpc-td-init
          image: gcr.io/trafficdirector-prod/td-grpc-bootstrap:0.16.0
          imagePullPolicy: Always
          args:
            - "--output=/tmp/bootstrap/td-grpc-bootstrap.json"
            - "--vpc-network-name=default"
            - "--xds-server-uri=trafficdirector.googleapis.com:443"
            - "--generate-mesh-id"
          volumeMounts:
            - mountPath: /tmp/bootstrap/
              name: grpc-td-conf
      volumes:
        - name: grpc-td-conf
          emptyDir:
            medium: Memory
EOF

kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: app1
spec:
  parentRefs:
  - name: helloworld
    kind: Service
    group: ""
  rules:
  - backendRefs:
    - name: helloworld
      port: 50051
EOF
```

### install the gRPC client

```
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: psm-grpc-client
  labels:
    app: psm-grpc-client
spec:
  replicas: 2
  selector:
    matchLabels:
      app: psm-grpc-client
  template:
    metadata:
      labels:
        app: psm-grpc-client
    spec:
      containers:
      - name: psm-grpc-client
        image: grpc/csm-o11y-example-python-client:latest
        imagePullPolicy: Always
        args:
          - "--target=xds:///helloworld.default.svc.cluster.local:50051"
        env:
          - name: GRPC_XDS_BOOTSTRAP
            value: "/tmp/grpc-xds/td-grpc-bootstrap.json"
          - name: CSM_WORKLOAD_NAME
            value: psm-grpc-client
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: NAMESPACE_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: CONTAINER_NAME
            value: psm-grpc-client
          - name: OTEL_RESOURCE_ATTRIBUTES
            value: k8s.pod.name=\$(POD_NAME),k8s.namespace.name=\$(NAMESPACE_NAME),k8s.container.name=\$(CONTAINER_NAME)
        volumeMounts:
          - mountPath: /tmp/grpc-xds/
            name: grpc-td-conf
            readOnly: true
      initContainers:
        - name: grpc-td-init
          image:  gcr.io/trafficdirector-prod/td-grpc-bootstrap:0.16.0
          imagePullPolicy: Always
          args:
            - "--output=/tmp/bootstrap/td-grpc-bootstrap.json"
            - "--vpc-network-name=default"
            - "--xds-server-uri=trafficdirector.googleapis.com:443"
            - "--generate-mesh-id"
          volumeMounts:
            - mountPath: /tmp/bootstrap/
              name: grpc-td-conf
      volumes:
        - name: grpc-td-conf
          emptyDir:
            medium: Memory

EOF
```

### enable observability

```
kubectl apply -f - <<EOF
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: psm-grpc-server-gmp
spec:
  selector:
    matchLabels:
      app: psm-grpc-server
  endpoints:
  - port: 9464
    interval: 10s
EOF

kubectl apply -f - <<EOF
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: psm-grpc-client-gmp
spec:
  selector:
    matchLabels:
      app: psm-grpc-client
  endpoints:
  - port: 9464
    interval: 10s
EOF
```

### deploy Hubble UI to observe traffic

```
kubectl apply -f dpv2_observability/

# access via Hubble CLI
alias hubble="kubectl exec -it -n gke-managed-dpv2-observability deployment/hubble-relay -c hubble-cli -- hubble"
hubble status
hubble observe

# access the Hubble UI
kubectl -n gke-managed-dpv2-observability port-forward service/hubble-ui 16100:80 --address='0.0.0.0'
```