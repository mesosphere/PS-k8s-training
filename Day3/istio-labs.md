## . Istio-Traffic-Management

### A. Request Routing

Ensure you default destination rules 
```bash
kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml
```
Wait a few seconds for the destination rules to propagate.

You can display the destination rules to propogate.
```bash
kubectl get destinationrules -o yaml
```
Run the following command to apply the virtual services:
```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```
Because configuration propagation is eventually consistent, wait a few seconds for the virtual services to take effect.

Display the defined routes with the following command:
```bash
kubectl get virtualservices -o yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: details
  ...
spec:
  hosts:
  - details
  http:
  - route:
    - destination:
        host: details
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productpage
  ...
spec:
  gateways:
  - bookinfo-gateway
  - mesh
  hosts:
  - productpage
  http:
  - route:
    - destination:
        host: productpage
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
  ...
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  ...
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
---
```
You have configured Istio to route to the v1 version of the Bookinfo microservices, most importantly the reviews service version 1.

Open the Bookinfo site in your browser. http://$GATEWAY_URL/productpage. Notice that the reviews part of the page displays with no rating stars, no matter how many times you refresh. This is because you configured Istio to route all traffic for the reviews service to the version reviews:v1 and this version of the service does not access the star ratings service.

You have successfully accomplished the first part of this task: route traffic to one version of a service.

Next, you will change the route configuration so that all traffic from a specific user is routed to a specific service version. In this case, all traffic from a user named Jason will be routed to the service reviews:v2.

Note that Istio doesn’t have any special, built-in understanding of user identity. This example is enabled by the fact that the productpage service adds a custom end-user header to all outbound HTTP requests to the reviews service.

Remember, reviews:v2 is the version that includes the star ratings feature.

Run the following command to enable user-based routing:
```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```
Confirm the rule is created:
```bash
kubectl get virtualservice reviews -o yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  ...
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
```
On the /productpage of the Bookinfo app, log in as user jason.

Refresh the browser. What do you see? The star ratings appear next to each review.

Log in as another user (pick any name you wish).

Refresh the browser. Now the stars are gone. This is because traffic is routed to reviews:v1 for all users except Jason.

You have successfully configured Istio to route traffic based on user identity.

#### Clean up 
```bash
kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

### B. Traffic Shifting
This task shows you how to gradually migrate traffic from one version of a microservice to another. For example, you might migrate traffic from an older version to a new version.

A common use case is to migrate traffic gradually from one version of a microservice to another. In Istio, you accomplish this goal by configuring a sequence of rules that route a percentage of traffic to one service or another. In this task, you will send 50% of traffic to reviews:v1 and 50% to reviews:v3. Then, you will complete the migration by sending 100% of traffic to reviews:v3.


#### Apply weight-based routing
If you haven’t already applied destination rules, follow the instructions in Apply Default Destination Rules.
To get started, run this command to route all traffic to the v1 version of each microservice.
```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

Open the Bookinfo site in your browser. The URL is http://$GATEWAY_URL/productpage, where $GATEWAY_URL is the External IP address of the ingress, as explained in the Bookinfo doc.

Notice that the reviews part of the page displays with no rating stars, no matter how many times you refresh. This is because you configured Istio to route all traffic for the reviews service to the version reviews:v1 and this version of the service does not access the star ratings service.

Transfer 50% of the traffic from reviews:v1 to reviews:v3 with the following command:
```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
```

Wait a few seconds for the new rules to propagate.

Confirm the rule was replaced:
```bash
kubectl get virtualservice reviews -o yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  ...
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 50
    - destination:
        host: reviews
        subset: v3
      weight: 50
```
Refresh the /productpage in your browser and you now see red colored star ratings approximately 50% of the time. This is because the v3 version of reviews accesses the star ratings service, but the v1 version does not.

With the current Envoy sidecar implementation, you may need to refresh the /productpage many times –perhaps 15 or more–to see the proper distribution. You can modify the rules to route 90% of the traffic to v3 to see red stars more often.
Assuming you decide that the reviews:v3 microservice is stable, you can route 100% of the traffic to reviews:v3 by applying this virtual service:
```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-v3.yaml
```
Now when you refresh the /productpage you will always see book reviews with red colored star ratings for each review.
#### Understanding what happened
In this task you migrated traffic from an old to new version of the reviews service using Istio’s weighted routing feature. Note that this is very different than doing version migration using the deployment features of container orchestration platforms, which use instance scaling to manage the traffic.

With Istio, you can allow the two versions of the reviews service to scale up and down independently, without affecting the traffic distribution between them.

For more information about version routing with autoscaling, check out the blog article Canary Deployments using Istio.
### C. Request Timeouts 

A timeout for http requests can be specified using the timeout field of the route rule. By default, the timeout is disabled, but in this task you override the reviews service timeout to 1 second. To see its effect, however, you also introduce an artificial 2 second delay in calls to the ratings service.

Route requests to v2 of the reviews service, i.e., a version that calls the ratings service:
```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v2
EOF
```

Add a 2 second delay to calls to the ratings service:
```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percent: 100
        fixedDelay: 2s
    route:
    - destination:
        host: ratings
        subset: v1
EOF
```

Open the Bookinfo URL http://$GATEWAY_URL/productpage in your browser.

You should see the Bookinfo application working normally (with ratings stars displayed), but there is a 2 second delay whenever you refresh the page.

Now add a half second request timeout for calls to the reviews service:
```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v2
    timeout: 0.5s
EOF
```
Refresh the Bookinfo web page.

You should now see that it returns in about 1 second, instead of 2, and the reviews are unavailable.
```
The reason that the response takes 1 second, even though the timeout is configured at half a second, is because there is a hard-coded retry in the productpage service, so it calls the timing out reviews service twice before returning.
```
#### Understanding what happened
In this task, you used Istio to set the request timeout for calls to the reviews microservice to half a second. By default the request timeout is disabled. Since the reviews service subsequently calls the ratings service when handling requests, you used Istio to inject a 2 second delay in calls to ratings to cause the reviews service to take longer than half a second to complete and consequently you could see the timeout in action.

You observed that instead of displaying reviews, the Bookinfo product page (which calls the reviews service to populate the page) displayed the message: Sorry, product reviews are currently unavailable for this book. This was the result of it receiving the timeout error from the reviews service.

If you examine the fault injection task, you’ll find out that the productpage microservice also has its own application-level timeout (3 seconds) for calls to the reviews microservice. Notice that in this task you used an Istio route rule to set the timeout to half a second. Had you instead set the timeout to something greater than 3 seconds (such as 4 seconds) the timeout would have had no effect since the more restrictive of the two takes precedence. More details can be found here.

One more thing to note about timeouts in Istio is that in addition to overriding them in route rules, as you did in this task, they can also be overridden on a per-request basis if the application adds an x-envoy-upstream-rq-timeout-ms header on outbound requests. In the header, the timeout is specified in milliseconds instead of seconds.

#### Clean up
```bash
kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

### D. Visualizing Mesh
This task shows you how to visualize different aspects of your Istio mesh.

As part of this task, you install the Kiali add-on and use the web-based graphical user interface to view service graphs of the mesh and your Istio configuration objects. Lastly, you use the Kiali Public API to generate graph data in the form of consumable JSON.
```
This task does not cover all of the features provided by Kiali. To learn about the full set of features it 
supports, see the Kiali website.
```


#### Generating a service graph
To verify the service is running in your cluster, run the following command:
```bash
$ kubectl -n istio-system get svc kiali
```
To determine the Bookinfo URL, follow the instructions to determine the Bookinfo ingress GATEWAY_URL.

To send traffic to the mesh, you have three options

Visit http://$GATEWAY_URL/productpage in your web browser
Use the following command multiple times:
```bash
curl http://$GATEWAY_URL/productpage
```
If you installed the watch command in your system, send requests continually with:
```bash
watch -n 1 curl -o /dev/null -s -w %{http_code} $GATEWAY_URL/productpage
```
To open the Kiali UI, execute the following command in your Kubernetes environment:
```bash
$ istioctl dashboard kiali
```
View the overview of your mesh in the Overview page that appears immediately after you log in. The Overview page displays all the namespaces that have services in your mesh. The following screenshot shows a similar page:
![alt text](https://istio.io/docs/tasks/observability/kiali/kiali-overview.png)

To view a namespace graph, click on the bookinfo graph icon in the Bookinfo namespace card. The graph icon is in the lower left of the namespace card and looks like a connected group of circles. The page looks similar to:
![alt text](https://istio.io/docs/tasks/observability/kiali/kiali-graph.png)

To view a summary of metrics, select any node or edge in the graph to display its metric details in the summary details panel on the right.

To view your service mesh using different graph types, select a graph type from the Graph Type drop down menu. There are several graph types to choose from: *App, Versioned App, Workload, Service.*
* The App graph type aggregates all versions of an app into a single graph node. The following example shows a single reviews node representing the three versions of the reviews app.
![alt text](https://istio.io/docs/tasks/observability/kiali/kiali-app.png)


* The Versioned App graph type shows a node for each version of an app, but all versions of a particular app are grouped together. The following example shows the reviews group box that contains the three nodes that represents the three versions of the reviews app.
![alt text](https://istio.io/docs/tasks/observability/kiali/kiali-versionedapp.png)

* The Workload graph type shows a node for each workload in your service mesh. This graph type does not require you to use the app and version labels so if you opt to not use those labels on your components, this is the graph type you will use.
![alt text](https://istio.io/docs/tasks/observability/kiali/kiali-workload.png)

* The Service graph type shows a node for each service in your mesh but excludes all apps and workloads from the graph.
![alt text](https://istio.io/docs/tasks/observability/kiali/kiali-services.png)

#### About Kiali
To generate JSON files representing the graphs and other metrics, health, and configuration information, you can access the Kiali Public API. For example, point your browser to $KIALI_URL/api/namespaces/graph?namespaces=bookinfo&graphType=app to get the JSON representation of your graph using the app graph type.

The Kiali Public API is built on top of Prometheus queries and depends on the standard Istio metric configuration. It also makes Kubernetes API calls to obtain additional details about your services. For the best experience using Kiali, use the metadata labels app and version on your application components. As a template, the Bookinfo sample application follows this convention.