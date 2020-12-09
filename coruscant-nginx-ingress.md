Setting up ExternalDNS on GKE with nginx-ingress-controller
This tutorial describes how to setup ExternalDNS for usage within a GKE cluster that doesn't make use of Google's default ingress controller but rather uses nginx-ingress-controller for that task.

Set up your environment
Setup your environment to work with Google Cloud Platform. Fill in your values as needed, e.g. target project.

$ gcloud config set project "zalando-external-dns-test"
$ gcloud config set compute/region "europe-west1"
$ gcloud config set compute/zone "europe-west1-d"

GKE Node Scopes
The following instructions use instance scopes to provide ExternalDNS with the permissions it needs to manage DNS records. Note that since these permissions are associated with the instance, all pods in the cluster will also have these permissions. As such, this approach is not suitable for anything but testing environments.

Create a GKE cluster without using the default ingress controller.

$ gcloud container clusters create "external-dns" \
    --num-nodes 1 \
    --scopes "https://www.googleapis.com/auth/ndev.clouddns.readwrite"
Create a DNS zone which will contain the managed DNS records.

$ gcloud dns managed-zones create "external-dns-test-gcp-jzaninidemo-com" \
    --dns-name "external-dns-test.gcp.jzaninidemo.com" \
    --description "Automatically managed zone by ExternalDNS"
Make a note of the nameservers that were assigned to your new zone.

$ gcloud dns record-sets list \
    --zone "external-dns-test-gcp-jzaninidemo-com" \
    --name "external-dns-test.gcp.jzaninidemo.com" \
    --type NS
NAME                             TYPE  TTL    DATA
external-dns-test.gcp.zalan.do.  NS    21600  ns-cloud-e1.googledomains.com.,ns-cloud-e2.googledomains.com.,ns-cloud-e3.googledomains.com.,ns-cloud-e4.googledomains.com.

In this case it's ns-cloud-{e1-e4}.googledomains.com. but your's could slightly differ, e.g. {a1-a4}, {b1-b4} etc.

Tell the parent zone where to find the DNS records for this zone by adding the corresponding NS records there. Assuming the parent zone is "gcp-zalan-do" and the domain is "gcp.zalan.do" and that it's also hosted at Google we would do the following.

gcloud dns record-sets transaction start --zone "gcp-jzaninidemo-com"
gcloud dns record-sets transaction add ns-cloud-b{1..4}.googledomains.com. \
  --name "external-dns-test.gcp.jzaninidemo.com." --ttl 300 --type NS --zone "gcp-jzaninidemo-com"
gcloud dns record-sets transaction execute --zone "gcp-jzaninidemo-com"

Connect your kubectl client to the cluster you just created and bind your GCP user to the cluster admin role in Kubernetes.

$ gcloud container clusters get-credentials "external-dns"
$ kubectl create clusterrolebinding cluster-admin-me \
    --clusterrole=cluster-admin --user="$(gcloud config get-value account)"
Deploy the nginx ingress controller
First, you need to deploy the nginx-based ingress controller. It can be deployed in at least two modes: Leveraging a Layer 4 load balancer in front of the nginx proxies or directly targeting pods with hostPorts on your worker nodes. ExternalDNS doesn't really care and supports both modes.

Default Backend
The nginx controller uses a default backend that it serves when no Ingress rule matches. This is a separate Service that can be picked by you. We'll use the default backend that's used by other ingress controllers for that matter. Apply the following manifests to your cluster to deploy the default backend.

apiVersion: v1
kind: Service
metadata:
  name: default-http-backend
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: default-http-backend

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: default-http-backend
spec:
  selector:
    matchLabels:
      app: default-http-backend
  template:
    metadata:
      labels:
        app: default-http-backend
    spec:
      containers:
      - name: default-http-backend
        image: gcr.io/google_containers/defaultbackend:1.3
Without a separate TCP load balancer
By default, the controller will update your Ingress objects with the public IPs of the nodes running your nginx controller instances. You should run multiple instances in case of pod or node failure. The controller will do leader election and will put multiple IPs as targets in your Ingress objects in that case. It could also make sense to run it as a DaemonSet. However, we'll just run a single replica. You have to open the respective ports on all of your worker nodes to allow nginx to receive traffic.

$ gcloud compute firewall-rules create "allow-http" --allow tcp:80 --source-ranges "0.0.0.0/0" --target-tags "gke-external-dns-9488ba14-node"
$ gcloud compute firewall-rules create "allow-https" --allow tcp:443 --source-ranges "0.0.0.0/0" --target-tags "gke-external-dns-9488ba14-node"
Change --target-tags to the corresponding tags of your nodes. You can find them by describing your instances or by looking at the default firewall rules created by GKE for your cluster.

Apply the following manifests to your cluster to deploy the nginx-based ingress controller. Note, how it receives a reference to the default backend's Service and that it listens on hostPorts. (You may have to use hostNetwork: true as well.)

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
spec:
  selector:
    matchLabels:
      app: nginx-ingress-controller
  template:
    metadata:
      labels:
        app: nginx-ingress-controller
    spec:
      containers:
      - name: nginx-ingress-controller
        image: gcr.io/google_containers/nginx-ingress-controller:0.9.0-beta.3
        args:
        - /nginx-ingress-controller
        - --default-backend-service=default/default-http-backend
        env:
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
        ports:
        - containerPort: 80
          hostPort: 80
        - containerPort: 443
          hostPort: 443
With a separate TCP load balancer
However, you can also have the ingress controller proxied by a Kubernetes Service. This will instruct the controller to populate this Service's external IP as the external IP of the Ingress. This exposes the nginx proxies via a Layer 4 load balancer (type=LoadBalancer) which is more reliable than the other method. With that approach, you can run as many nginx proxy instances on your cluster as you like or have them autoscaled. This is the preferred way of running the nginx controller.

Apply the following manifests to your cluster. Note, how the controller is receiving an additional flag telling it which Service it should treat as its public endpoint and how it doesn't need hostPorts anymore.

Apply the following manifests to run the controller in this mode.

apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress-controller
spec:
  type: LoadBalancer
  ports:
  - name: http
    port: 80
    targetPort: 80
  - name: https
    port: 443
    targetPort: 443
  selector:
    app: nginx-ingress-controller

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
spec:
  selector:
    matchLabels:
      app: nginx-ingress-controller
  template:
    metadata:
      labels:
        app: nginx-ingress-controller
    spec:
      containers:
      - name: nginx-ingress-controller
        image: gcr.io/google_containers/nginx-ingress-controller:0.9.0-beta.3
        args:
        - /nginx-ingress-controller
        - --default-backend-service=default/default-http-backend
        - --publish-service=default/nginx-ingress-controller
        env:
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
        ports:
        - containerPort: 80
        - containerPort: 443
Deploy ExternalDNS
Apply the following manifest file to deploy ExternalDNS.

apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
---
apiVersion: rbac.authorization.k8s.io/v1beta1
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
apiVersion: rbac.authorization.k8s.io/v1beta1
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
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: k8s.gcr.io/external-dns/external-dns:v0.7.3
        args:
        - --source=ingress
        - --domain-filter=external-dns-test.gcp.zalan.do
        - --provider=google
        - --google-project=zalando-external-dns-test
        - --registry=txt
        - --txt-owner-id=my-identifier
Use --dry-run if you want to be extra careful on the first run. Note, that you will not see any records created when you are running in dry-run mode. You can, however, inspect the logs and watch what would have been done.

Deploy a sample application
Create the following sample application to test that ExternalDNS works.

apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: nginx
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: via-ingress.external-dns-test.gcp.zalan.do
    http:
      paths:
      - backend:
          serviceName: nginx
          servicePort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx

---

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
After roughly two minutes check that a corresponding DNS record for your Ingress was created.

$ gcloud dns record-sets list \
    --zone "external-dns-test-gcp-zalan-do" \
    --name "via-ingress.external-dns-test.gcp.zalan.do." \
    --type A
NAME                                         TYPE  TTL  DATA
via-ingress.external-dns-test.gcp.zalan.do.  A     300  35.187.1.246
Let's check that we can resolve this DNS name as well.

dig +short @ns-cloud-e1.googledomains.com. via-ingress.external-dns-test.gcp.zalan.do.
35.187.1.246
Try with curl as well.

$ curl via-ingress.external-dns-test.gcp.zalan.do
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</head>
<body>
...
</body>
</html>

