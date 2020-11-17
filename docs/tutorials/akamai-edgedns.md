# Setting up ExternalDNS for Services on Akamai Edge DNS

## Prerequisites

Akamai Edge DNS (formally known as FastDNS) provider support was first released in External-DNS v0.5.18

### Zones

The Akamai Edge DNS provider requires that the zones in which you want to manage kubernetes service endpoints already exist and are configured correctly. The provider does not add, remove or configure new zones in anyway. Edge DNS zones can be managed thru the [Akamai Control Center] (https://control.akamai.com)

### Akamai Edge DNS Authentication

The Akamai Edge DNS provider requires valid Akamai API authentication credentials to access zones and manage associated DNS records. 

Credentials can be provided to the provider either directly by key or indirectly via a file. The Akamai credential keys and mappings to the Akamai provider utilizing different presentation methods are:

| Akamai | Provider  | Operating | Kubernetes |
| Auth Key | Cmd Line | Environment | ConfigMap |
| -------- | -------- | ------------------ | ----------- |
| host | akamai-serviceconsumerdomain | EXTERNAL_DNS_AKAMAI_SERVICECONSUMERDOMAIN ||
| access_oken | akamai-access-token | EXTERNAL_DNS_AKAMAI_ACCESS_TOKEN ||
| client_token | akamai-client-token  | EXTERNAL_DNS_AKAMAI_CLIENT_TOKEN ||
| client-secret | akamai-client-secret | EXTERNAL_DNS_AKAMAI_CLIENT_SECRET ||

In addition to specfying auth credentials individually, the credentials my be referenced indirectly by using the Akamai Edgegrid .edgerc file convention.

| Provider  | Operating | Kubernetes |
| Cmd Line | Environment | ConfigMap |
| -------- |  ------------------ | ----------- |
| akamai-edgerc-path | EXTERNAL_DNS_AKAMAI_EDGERC_PATH ||
| akamai-edgerc-section | EXTERNAL_DNS_AKAMAI_EDGERC_SECTION ||

[Akamai API Authentication](https://developer.akamai.com/getting-started/edgegrid) provides an overview and further information pertaining to the generation of auth credentials for API base applications and tools.

The following example defines and references a Kubernetes ConfigMap secret is used by referencing the secret and its keys in the env section of the deployment.


## Deploy ExternalDNS

An operational ExternalDNS deployment consists of an ExternalDNS container and service. The following sections demonstrate the ConfigMap objects that would make up an example functional external DNS kubernetes configuration.

Connect your `kubectl` client to the cluster with which you want to test ExternalDNS, and then apply one of the following manifest files for deployment:

### Manifest (for clusters without RBAC enabled)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      containers:
      - name: external-dns
        image: k8s.gcr.io/external-dns/external-dns:v0.7.3
        args:
        - --source=service  # or ingress or both
        - --provider=akamai
        - --domain-filter=example.com
        # zone-id-filter may be specified as well to filter on contract ID
        - --registry=txt
        - --txt-owner-id={{ owner-id-for-this-external-dns }}
        env:
        - name: EXTERNAL_DNS_AKAMAI_SERVICECONSUMERDOMAIN
          valueFrom:
            secretKeyRef:
              name: external-dns
              key: EXTERNAL_DNS_AKAMAI_SERVICECONSUMERDOMAIN
        - name: EXTERNAL_DNS_AKAMAI_CLIENT_TOKEN
          valueFrom:
            secretKeyRef:
              name: external-dns
              key: EXTERNAL_DNS_AKAMAI_CLIENT_TOKEN
        - name: EXTERNAL_DNS_AKAMAI_CLIENT_SECRET
          valueFrom:
            secretKeyRef:
              name: external-dns
              key: EXTERNAL_DNS_AKAMAI_CLIENT_SECRET
        - name: EXTERNAL_DNS_AKAMAI_ACCESS_TOKEN
          valueFrom:
            secretKeyRef:
              name: external-dns
              key: EXTERNAL_DNS_AKAMAI_ACCESS_TOKEN
```

### Manifest (for clusters with RBAC enabled)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
rules:
- apiGroups: [""]
  resources: ["services","endpoints","pods"]
  verbs: ["get","watch","list"]
- apiGroups: ["extensions","networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get","watch","list"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
- kind: ServiceAccount
  name: external-dns
  namespace: default
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      containers:
      - name: external-dns
        image: k8s.gcr.io/external-dns/external-dns:v0.7.3
        args:
        - --source=service  # or ingress or both
        - --provider=akamai
        - --domain-filter=example.com
        # zone-id-filter may be specified as well to filter on contract ID
        - --registry=txt
        - --txt-owner-id={{ owner-id-for-this-external-dns }}
        env:
        - name: EXTERNAL_DNS_AKAMAI_SERVICECONSUMERDOMAIN
          valueFrom:
            secretKeyRef:
              name: external-dns
              key: EXTERNAL_DNS_AKAMAI_SERVICECONSUMERDOMAIN
        - name: EXTERNAL_DNS_AKAMAI_CLIENT_TOKEN
          valueFrom:
            secretKeyRef:
              name: external-dns
              key: EXTERNAL_DNS_AKAMAI_CLIENT_TOKEN
        - name: EXTERNAL_DNS_AKAMAI_CLIENT_SECRET
          valueFrom:
            secretKeyRef:
              name: external-dns
              key: EXTERNAL_DNS_AKAMAI_CLIENT_SECRET
        - name: EXTERNAL_DNS_AKAMAI_ACCESS_TOKEN
          valueFrom:
            secretKeyRef:
              name: external-dns
              key: EXTERNAL_DNS_AKAMAI_ACCESS_TOKEN
```

Create the deployment for ExternalDNS:

```
$ kubectl create -f externaldns.yaml
```

## Deploying an Nginx Service

Create a service file called 'nginx.yaml' with the following contents:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    external-dns.alpha.kubernetes.io/hostname: nginx.example.com
    external-dns.alpha.kubernetes.io/ttl: "600" #optional
spec:
  selector:
    app: nginx
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Create the deployment, service and ingress object:

```
$ kubectl create -f nginx.yaml
```

## Verify Akamai Edge DNS Records

It is recommended to wait 3-5 minutes before validating the records to allow the record changes to propagate to all the Akamai name servers worldwide.

The records can be validated using the [Akamai Control Center](http://control.akamai.com) or by executing a dig, nslookup or similar DNS command.
 
## Cleanup

Once you successfully configure and verify record management via ExternalDNS, you can delete the tutorial's example:

```
$ kubectl delete -f nginx.yaml
$ kubectl delete -f externaldns.yaml
```
