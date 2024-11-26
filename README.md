
# Beron's Istio Lab & Configuration

This Repository contains the most up to date version of my Istio project Lab.

The purpose of this project was to create a repeatable and flexible environment to test and familiarize myself with the architecture and most prominent features of the Istio operator. I found in my studies and experiments, that though at many points it's configuration can be rather tricky, the advantages of running Istio in production outweigh the mental labor of it's initial configuration.
Due to the way its custom resources reference other objects it's imperative that you maintain a consistent understandable naming and labeling convention as these will be utilized repetitiously throughout configuration and modification.  Due to it's complex nature, I think it would be best to give a brief explanation of what Istio is and what it does, and then devote the rest of this document to the deployment steps of this repo's content.

## What is Istio?

Istio is an open-source **service mesh** that simplifies how microservices in a Kubernetes cluster communicate securely and efficiently. It manages:

1. **Traffic Control**: Route and balance traffic between services.
2. **Security**: Encrypt communication between services using mTLS.
3. **Observability**: Monitor and visualize service communication and performance.
4. **Resilience**: Add retries, timeouts, and failover policies to improve service reliability.

Istio works by deploying a control plane to manage policies and sidecar proxies in each service pod to handle network traffic. Think of it as a layer that takes care of all networking concerns so developers can focus on their applications.

When deploying pods within an Istio-enabled namespace, Istio automatically injects sidecar proxies into each pod. This automation simplifies management, as all traffic between services is transparently handled through these sidecars.

All traffic managed by Istio flows through the sidecar proxies, creating a single point for administering policies and enabling observability. For example, if you want to encrypt all traffic within a namespace, you can enable mTLS (mutual TLS). Instead of configuring each application individually to support encrypted communication, Istio handles the encryption through its sidecars, removing the complexity from developers and ensuring secure communication seamlessly.

As you’ll see later in this document, Istio has many more fascinating and unique capabilities to explore.

---
#### Lab Deployment

As mentioned earlier, the best way to understand how Istio works is by deploying it and learning the significance of each step. In this walkthrough, I’ll provide a description of the key characteristics for each YAML file we deploy. 

##### *Prerequisite*  
In order to do the minimum specification for HTTP. You do not need anything besides a Kubernetes cluster to begin working with this. 

However if you desire to enable HTTPS you will need a google registered domain to follow the last section of this guide. 

---
## Section 1 *Operator & Control Plane Namespace*

First we create a namespace for our Istio control-plane and tools to live in.
`kubectl create namespace istio-system`

Next we verify that our terminal is in the top level of the directory after git-cloning the repository. When you use `ls` in your terminal you should see both the certs & the yaml folder. 

Now we will enter the following command:
```python
kubectl create secret generic cacerts -n istio-system \
--from-file=certs/ca-cert.pem \
--from-file=certs/ca-key.pem \
--from-file=certs/root-cert.pem \
--from-file=certs/cert-chain.pem
```
This command will save the contents of the certs folder as a Kubernetes secret. A secret within Kubernetes is akin to the secret manager in GCP. Just exclusive the the Kubernetes environment. 

When the operator is deployed it will know to use this secret to access the certifications needed to deploy Istio. 


Now change directories into the `yaml` folder.

And we will install the operator/control plane using the 001-yaml template.
`istioctl install -y -f 001-istio-operator.yaml -n istio-system --revision prod`
*Notice how we tag the operator/control-plane with the* `revision prod`

Get the pods from the istio-system namespace to confirm all is well and to check your work so far:
`kubectl get pods -n istio-system`

If the operator has a healthy and running status we can continue to the next section.

---
## Section 2 *Webserver deployments*
---

Now let us create the Istio managed namespace to host our demo web applications. 

From the top of the YAML folder create the namespace:
`kubectl apply -f 000-app-namespace.yaml`

And now use the `--revision prod` label tagged to the control plane from earlier to mark our namespace for istio management. 
`kubectl label namespace lizzoslunch istio.io/rev=prod`
*Notice how by adding the matching label to our namespace, istio recognizes it as a resource to manage*


We're now ready to deploy our applications. Before we do so, I want to highlight a few things from the YAML. 

cd into `test-app` and pick an application and look at it's 001-deployment yaml. 

You'll notice these applications are stateful sets in line with my most recent secure updates. 

Pay attention to the following:

Notice how this application sports the additional label of `istio: monitor` 
- This is to identify our pods for monitoring through Prometheus
```python
spec:
  replicas: 1
  selector:
    matchLabels:
      app: type-a
  updateStrategy:
    type: RollingUpdate # When this configuration is re-applied, pods are gracefully updated with new configuration.
  template:
    metadata:
      labels:
        app: type-a
        istio: monitor
```

And Here we specify ports for our application listener, and an another called `http-envoy-prom`
- The additional port is uses by the istio sidecars to expose their metrics,  which Prometheus also scrapes by default—a beautiful integration. By exposing this port within our containers, we enable Prometheus to monitor traffic flowing through the service mesh.
```python
     #>>> PORTS 
        ports:
        - name: http 
          containerPort: 5000 # application listenr
        - name: http-envoy-prom # prom istio port
          containerPort: 15090 # prom-monitoring default
```

Now change directories to the top of `test-app` and run the following commands in the same order.

Check each application is running before deploying the next

1. kubectl apply -f type-a
2. kubectl apply -f type-b
3. kubectl apply -f type-c

Once all of the pods in the `lizzoslunch` namespace are confirmed as healthy we can continue onto the next step. 

*Notice how when running `get pods` we have 2 containers instead of the 1 as outlined in the manifest. This is due to the sidecar container created and attached by istio.* 


---
## Section 3 *Gateway installation and settings*
---

This section can get confusing near the end but stay with me!

Change directories back to the top of the `yaml` folder. And install the global ingress gateway:
`istioctl install -y -f 002-global-ingress-gateway.yaml --revision prod`

This will create an external facing gateway within the `istio-system `with a public IP address
After creation view the available services to get our ip address and save this for the next steps.
`kubectl get svc -n istio-system`

With our IP address in tow and in mind Lets edit and take a look at the `003-gateway-settings.yaml`

A Brief description on each resource:
##### Gateway Configuration

- **Gateway**:
    - **Purpose**: Configures an  internal entry point for inbound HTTP (port 80) or HTTPS (port 443) traffic into the service mesh.
    - **Key Feature**: Tied to the `global` ingress configuration, enabling external access to the mesh via the specified domains or IPs.
    *Leave https commented until you have a certified domain*
```python
#>>>>>------------------GATEWAY-CONFIGURATION---------------------------------- 
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: lunar-gateway
  namespace: istio-system
spec:
  selector:
    ingress: global # Ties the gateway to the operator in istio-system
  servers:
#HTTP-PORT-80
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"  # Update to domain after creating an A-record berongcp.net or use * 
HTTPS-PORT-443
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "berongcp.net"  # Update to domain after creating an A-record berongcp.net
    tls:
      mode: SIMPLE
      credentialName: api-berongcp-net-crt # Secret created by the cert-manager
```

##### Horizontal Pod Autoscaler (HPA)

- **HorizontalPodAutoscaler**:
    - **Purpose**: Dynamically scales the amount of exterior ingress gateway instances based on CPU utilization in relation to traffic
    - **Key Feature**: Ensures high availability by scaling pods based on traffic demand.
```python
#>>> GATEWAY-AUTOSCALING-FOR-HIGH-AVAILABILITY
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: global-ingress-hpa
  namespace: istio-system
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: global-ingress  # Match the Deployment created by IstioOperator
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
--- 
```
###### Pod Disruption Budget (PDB)

- **PodDisruptionBudget**:
    - **Purpose**: Ensures that at least one pod from the `global-ingress` deployment is always available during voluntary disruptions (e.g., node maintenance).
    - **Key Feature**: Provides resiliency and minimizes downtime during updates or scaling.

###### VirtualService Configuration

- **VirtualService**:
    - **Purpose**: Routes incoming traffic from the `lunar-gateway` to specific services and subsets within the service mesh. Sort of like a URL map for istio.
    - **Key Feature**:
        - Routes based on URI prefixes (`/redhead`, `/milk`, `/italia`).
        - Supports traffic splitting between service subsets.
        - Allows weighted distribution of traffic for A/B testing or gradual rollouts.
        *Underneath hosts replace the parentheses with your gateway svc ip address*
        *Each application is represented with it's own route*
```python
#>>>>>-------------VIRTUAL-SERVICE/ROUTING-CONFIGURATION----------------------- 
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app-virtualservice
  namespace: lizzoslunch #
spec:
# >>> First update this field with the generated gateway IP. Then Update the host after creating an A-record berongcp.net
  hosts:
  - "FILL-IP-OR-DOMAIN-HERE" # LoadBalancer IP or berongcp.net 
  gateways:
  - istio-system/lunar-gateway
#>>> ROUTE-1-type-a-stable-subset
  http:
    - match: 
        - uri:
            prefix: /redhead 
      rewrite:  
        uri: "/"       
      route:
        - destination:
            host: type-a-service.lizzoslunch.svc.cluster.local #
            subset: stable #
#>>> ROUTE-2-type-b-stable-subset
    - match: 
        - uri:
            prefix: /milk
      rewrite:  
        uri: "/"         
      route:
        - destination:
            host: type-b-service.lizzoslunch.svc.cluster.local #
            subset: stable #
#>>> ROUTE-3-type-a-canary-subset
    - match: 
        - uri:
            prefix: /italia
      rewrite:  
        uri: "/"         
      route:
        - destination:
            host: type-a-service.lizzoslunch.svc.cluster.local #
            subset: canary #
#>>> ROUTE-0-DEFAULT-URL-MUST-BE-PLACED-LAST            
    - route:  
        - destination:
            host: type-a-service.lizzoslunch.svc.cluster.local #
            subset: stable #
          weight: 25
        - destination:
            host: type-a-service.lizzoslunch.svc.cluster.local #
            subset: canary #
          weight: 25
        - destination:
            host: type-b-service.lizzoslunch.svc.cluster.local #
            subset: stable #
          weight: 50
```
###### Routing and Traffic Management

The schema for the routes within the virtual service are devised from our destination rule resources. These are located in each application folder, But I will talk you through the below example. 

##### Destination Rule

- **Purpose**: Defines policies and subsets for routing traffic to specific versions or instances of a service.
- **Key Features**:
    - **Service Targeting**: Specifies the target service (`type-a-service.lizzoslunch.svc.cluster.local`) for which traffic policies apply.
    - **Subsets**:
        - **Stable**: Represents the default or production-ready version of the service (e.g., `app: type-a`).
        - **Canary**: Represents a specific version or experimental deployment of the service (e.g., `app: type-a` with `version: v1`).
    - **Traffic Control**: Used in conjunction with a VirtualService to control traffic distribution between subsets (e.g., for canary deployments or A/B testing).
```python
# >>> DESTINATION-RULE-EXAMPLE
 apiVersion: networking.istio.io/v1beta1
 kind: DestinationRule
 metadata:
   name: type-a
   namespace: lizzoslunch
 spec:
   host: type-a-service.lizzoslunch.svc.cluster.local #
   subsets:
     - name: stable
       labels:
         app: type-a
     - name: canary
       labels:
         app: type-a
         version: v1
```
*This routing connection between the virtual service and destination rule for type-a is what enables us to share services between multiple applications, and load balance/establish routes based on the labels exclusively*

As shown above each subset uses an exact match of the labels specified to create a path for routing. Thus enabling this configuration to manage the difference between application and it's service through use of it's labels.

Whew! That's alot. Hopefully I didn't lose you there

To wrap up this section make sure to update the quotes underneath `host` with your gateway's ip address. 

Then `kubectl apply -f 003-gateway-settings.yaml` and you should have access to all applications via the external IP. Refresh to see the load balancer distribute traffic.

---
## Section 4 *Monitoring with Prometheus, Grafana, and Kiali*
---

Now that you made it through the networking section and have an idea of how istio works, we ought to take advantage of it's features and begin monitoring our service mesh. 
This is where the labels and ports I pointed out on our stateful set will come into play. 

Our Prometheus instance will locate the istio containers through the namespace, the labels `istio : monitor`, and the identified Prometheus port of `http-envoy-prom` on our sidecar. 
```python
---
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: istio-pods
  namespace: monitoring
  labels:
    prometheus: main
spec:
  namespaceSelector: # < namespaces to allow prometheus to observe
    matchNames:
      - lizzoslunch # < Namespace for prometheus to monitor
  selector:
    matchLabels:
      istio: monitor # < labels on our applications
  podMetricsEndpoints:
    - port: http-envoy-prom # < reference to our portname
      path: stats/prometheus
```

Now we apply our monitoring configuration in a similar fashion as we've done previously with Theo.

cd into the top of `monitoring` folder, and run the following commands and wait for the completion of each. 
```python
kubectl create -f prometheus-operator-crds/

kubectl apply -f monitoring-ns.yaml

kubectl apply -R -f prometheus-operator

kubectl apply -f prometheus

kubectl apply -f grafana
```

Verify the Health of your monitoring resources. using `kubectl get pods -n monitoring` & `kubectl get pods -n app-g`

Now apply the pod monitor to begin scraping data with Prometheus.  
`kubectl apply -f istio-sidecars-pod-monitor.yaml`

And ingress into our Prometheus instance using port forwarding. Open up a new terminal, and type the following command. 
`kubectl port-forward svc/prometheus-operated 9090 -n monitoring`

Inside of your preferred web browser access Prometheus using the following URL: 
`http://localhost:9090`

And after a few moments and refreshes you should see data being scraped from your pods within the target menu. 

Now access your Grafana instance by finding it's ip:  `kubectl get svc -n app-g`
And create a new dashboard for Prometheus using it's cluster DNS name as a data source. 
`http://prometheus-operated.monitoring.svc.cluster.local:9090`

You are now monitoring your service mesh using Prometheus and Grafana! 
If you have my repository for a load tester you can see your traffic graphs using the following standard queries: 
``` python
istio_request_duration_milliseconds_bucket
istio_requests_total
scrape_duration_seconds
```


And for further automated visualizations of our mesh we can apply our Kiali folder and run a load test to watch traffic routing within our network live. 

`kubectl apply -f kiali/`
Port-forward for access 
`kubectl port-forward svc/kiali 20001 -n istio-system`
URL:
`http://localhost:20001`

And once inside in order to see the powerful visuals being generated, switch to the lizzoslunch namespace and activate a load test. You will then navigate to the graph page and see the magic take place. 

Kiali will visualize, and track successful, failed requests, and routing maneuvers. 
And It also allows you to to rewind and pause traffic activity over time, So you can see what happened during particular events by highlighting the period of time you wish to view. An extremely powerful tool!


The next section will Require a GCP Registered domain to use

---
## Section 5 *HTTPS using the Istio Gateway*
---

In order to enable HTTPS in my configuration we only have to make a few adjustments to our virtual service and employ a few new tools. Will will be making use of the cert-manager operator to generate certificates for an "A" record tied to our domain. The certificates will then be stored in the kubectl secrets manager and then used by our gateway to terminate incoming https encryption. 
*In my example I'll be using berongcp.net*

First Create an `A-Record` for your domain and tie it to the External IP address of your gateway. 
```python
gcloud dns record-sets create berongcp.net. \
  --type=A \
  --zone=berongcp-net \
  --ttl=300 \
  --rrdatas=34.32.11.171
```


Now update the following fields on the gateway with your domain: 

The hosts under port 80:
```python
#HTTP-PORT-80
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "berongcp.net"  # < Update Here with your domain
```



And the hosts under the virtual service:
```python
#>>>>>-------------VIRTUAL-SERVICE/ROUTING-CONFIGURATION----------------------- 
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app-virtualservice
  namespace: lizzoslunch #
spec:
  hosts:
  - "berongcp.net" # < update here
```
If you apply the Gateway file again, you'll notice that your domain will begin accepting unencrypted traffic on port 80. 

Now that we know our domain is working properly.

Lets download the CRD's for the cert-manager into our cluster:
```python
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml
```

After Successful completion, From our YAML folder let's apply:
`kubectl apply -f 004-clusterissuer.yaml`
This will install the istio iteration of the cert manager.

Using`kubectl get clusterissuer` we can confirm that the installation is running smoothly. 

Edit the `005-cert-manager.yaml` file to match the dns name, secret name, and metadata name to your values of choice:
```python
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-berongcp-net # < Change this
  namespace: istio-system
spec:
  secretName: api-berongcp-net-crt # < Change this
  dnsNames:
    - berongcp.net # < Change this
  issuerRef:
    name: production-cluster-issuer
    kind: ClusterIssuer
    group: cert-manager.io 
```

Now we apply the YAML:
`kubectl apply -f 005-cert-manager.yaml`

And if we have no errors, we will make one final edit to the 003 gateway file. 

Uncomment the https section and replace my domain with yours:
```python
#HTTPS-PORT-443
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "berongcp.net"  # Update to your domain
    tls:
      mode: SIMPLE
      credentialName: api-berongcp-net-crt # Change to your secret name
```

After applying the Gateway file for one last time HTTPS should now be working and tracking withing your istio configuration. 

If you managed to get this far, feel free to dm screenshots of your success, or if you have questions as you build through this process feel free to reach out to me as well. 

Thank you so much for taking the time to learn Istio with me!

---
