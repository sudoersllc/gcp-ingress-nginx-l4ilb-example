# gcp-nginx-ingress-l4ilb-example


> As of June 29, 2020,  [L7 Internal Load Balancing on GCP
](https://cloud.google.com/load-balancing/docs/l7-internal#shared_vpc) is a feature availble in the Alpha
release phase which requires the project to be whitelisted.

Additionally, this alpha release does not yet include L7 ILB launched and managed by GKE.

Because of this current limitation, a long standing architectural pattern is to be relied on: Terminate SSL upstream. 
Here, one will find that pattern which will allow for TLS Termination _inside_ of the GKE cluster via
[nginx-ingress](https://kubernetes.github.io/ingress-nginx/). 

```
+--------+        +--------------------------+          +--------------+
|L4 ILB  +--------+   https://nginx-ingress  +----------+ClusterIP Svc |
|        |        |                          |          |              |
+--------+        +--------------------------+          +--------------+
```


  Internal Load Balancer exposition to the shared VPC network is to be provided by the GA supported [L4 Internal Load Balancer](
https://cloud.google.com/load-balancing/docs/internal/). A request sent to this ILB would be, via forwarding rule, be forwarded 
to the regional internal backend service and onto the targets inside the gke nodepool via their NodePort exposition from the nginx-ingress.

![L4 ILB](https://cloud.google.com/load-balancing/images/ilb-high-level.svg)


This code sample assumes that you are bringing your own certificate. It assumes you bypass the CSR and/or interacting with PKI and you directly
store such sensitive files within a kubernetes secret object. For your safety, it also assumes that RBAC is an accepted concept and set up properly to 
discourage deny unintended actors access to such sensitive content. 

For self-signing in a testing context: you 
are picking this up for testing, [you can self-sign certs with the openssl cli tool,
](https://kubernetes.github.io/ingress-nginx/examples/PREREQUISITES/#tls-certificates) and then, you can import the resulting
ca, server key, (and optionally client key), [files as kubernetes secrets](
https://kubernetes.github.io/ingress-nginx/examples/auth/client-certs/#creating-certificate-secrets). 

When bringing certs, One can follow this [later process of importing key files into kubernetes secrets](
https://kubernetes.github.io/ingress-nginx/examples/auth/client-certs/#creating-certificate-secrets). 


Steps to get this working;

1. Choose or create a namespace
2. build and the `Dockerfile` and push to a registry that is of your choosing.
3. Update the `deploy/socat.deployment.yml` files image field so that it references your freshly built docker image
4. Deploy the socat `Deployment`
  a. `kubectl -n <namespace> apply -f deploy/socat.deployment.yml`
5. deploy the socat `Service`
  * a. `kubectl -n <namespace> apply -f deploy/socat.svc.yml`
  * b. this is the clusterIP service that is routeable only within the cluster. 
    The nginx-ingress will programatically add this service as the upstream target to reverse-proxy to.
6.  install + prepare helm 
  * install helm3 per the documentation - https://helm.sh/docs/intro/install/
  * add the stable ingress-nginx charts repo 
    * `helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx`
    * follow https://kubernetes.github.io/ingress-nginx/deploy/#using-helm
7. inspect the [values.yaml](values.yaml) as these are parameters to the ingress-nginx helm chart
8. deploy the nginx-ingress helm chat
    * `./helm_deploy_ingress.sh`
9. validate that the ingress-nginx deployed as intended
    * `kubectl -n <namespace> get deployments; kubectl -n <namespace> get hpa;  kubectl -n <namespace> get pods; kubectl -n <namespace> get svc`
    * in all of these places you should see running pods and services (depending on the values.yaml). inspect the service so that it is of type NodePort. Look for a horizontal_pod_autoscaler
10. Deploy the application ingress object
    * assuming you have already imported your certificate as a secret, now look at `deploy/socat-service.ing.yml` and
      ensure the secretName under the tls node matches the secret you created containing your tls cert
    * validate that the ingress-nginx acts as intended. One way of validating is by logging into a Pod within the cluster/namespace
      with `kubectl exec -it <podname> /bin/sh` and manually curling the nginx-endpoint. If all looks good, proceed.
12. Verify everything works, from inside the sVPC
     * `curl -vvv https://<yourEndpoint>` 





