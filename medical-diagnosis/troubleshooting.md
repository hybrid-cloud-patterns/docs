---
layout: default
title: Troubleshooting Guide
grand_parent: Patterns
parent: Medical Diagnosis
nav_order: 4
---
## Contents
{: .no_toc}

1. TOC
{:toc}

### Understanding the Makefile
The Makefile is the entrypoint for the pattern. We use the Makefile to bootstrap the pattern 
to the cluster. After the inital bootstrapping of the pattern, the Makefile isn't required as much anymore. Sometimes it is necessary to run a `make upgrade` which allows us to refresh the bootstrap resources without having to tear down the cluster.

#### make install / make deploy
Executing `make install` within the pattern application will trigger a `make deploy` from `<pattern_directory>/common`. This initializes the **common** components of the pattern framework and will install a helm chart in the `default` namespace. At this point cluster services such as **Red Hat Advanced Cluster Management** and **OpenShift Gitops** are deployed. 

Once **common** completes, the remaining tasks within the `make install` target will execute. 

#### make vault-init / make load-secrets
This pattern is integrated with **HashiCorp Vault** and **External Secrets** services for secrets management within the cluster. These targets install vault from a Helm chart and the load the secret `(values-secret.yaml)` you created during [Getting Started](../getting-started#preparation). 

If **values-secret.yaml** does not exist, make will exit with an error saying so. Furthermore, if the **values-secret.yaml** file does exist but is improperly formatted, ansible will exit with an error about being improperly formatted. If you are not sure how format the secret, please refer to [Getting Started](../getting-started#preparation).

#### make bootstrap / make upgrade
`make bootstrap` is the target used for deploying the application specific components of the pattern. It is the final step in the inital `make install` target. Running `make bootstrap` directly should typically not be necessary, instead you are encouraged to run `make upgrade`. 

Generally, executing `make upgrade` should only be required when something goes wrong with the application pattern deployment. For instance, if a value was missed, and the chart wasn't rendered correctly, executing `make upgrade` after fixing the value would be necessary.

If you have any further questions, please, feel free to review the `Makefile` for the **common** and **Medical Diagnosis** components. It is located in `common/Makefile` and `./Makefile` respectively. 

### Troubleshooting the Pattern Deployment
Occasionally the pattern will encounter issues during the deployment. This can happen for any number of reasons, but most often it is because of either a change within the operator itself or something has changed/happened to the Operator Lifecycle Manager (OLM) which determines which operators are available in the operator catalog. Generally, when an issue occurs with the OLM, the operator is unavailable for installation. To ensure that the operator is in the catalog:

```sh
oc get packagemanifests | grep <operator-name>
```
When an issue occurs with the operator itself you can verify the status of the `subscription` and make sure that there are no warnings.An additional option is to log into the OpenShift Console, click on Operators, and check the status of the operator.

> **Use the grafana dashboard to assist with debugging and identifying the issue**

---

**Problem**: No information is being processed in the dashboard

**Solution**: Most often this is due to the image-generator deploymentConfig needing to be scaled up. The image-generator by design is **scaled to 0**; 

```sh
oc scale -n xraylab-1 dc/image-generator --replicas=1
```

Or open the openshift-console, click on workloads, then click deploymentConfigs, click image-generator, and scale the pod to 1 or more. 

---

**Problem**: When browsing to the **xraylab** grafana dashboard and there are no images in the right-pane, only a security warning. 

**Solution**: The certificates for the openshift cluster are untrusted by your system. The easiest way to solve this is to open a browser and go to the s3-rgw route (oc get route -n openshift-storage), then acknowledge and accept the security warning. 

---

**Problem**: In the dashboard there is no metrics data available.

**Solution**: There is likely something wrong with the Prometheus DataSource for the grafana dashboard. You can check the status of the datasource by executing the following:

```sh
oc get grafanadatasources -n xraylab-1
```

Ensure that the prometheus datasource exists and that the status is available. This could potentially be the token from the service account (**grafana-serviceaccount**) that is provided to the datasource as a bearer token.

---

**Problem**: The dashboard is showing red in the corners of the dashboard panes.

**Solution**: This is most likely due to the **xraylab** database not being available or misconfigured. Please check the database and ensure that it is functioning properly.

---

**Problem**: The image-generator is scaled correctly, but nothing is happening in the dashboard.

**Solution**: This could be that the sererless eventing function isn't picking up the notifications from ODF and therefore, not triggering the knative-serving function to scale up. In this situation there are a number of things to check, the first thing is to check the logs of the `rgw` pod in the `openshift-storage` namespace. 

```sh
oc logs -n openshift-storage -f <pod> -c rgw
```
You should see the `PUT` statement with a status code of `200`

Next ensure that the `kafkasource` and `kafkservice` and `kafka topic` resources have been created:
```sh
oc get -n xraylab-1 kafkasource

oc get -n xraylab-1 kservice

oc get -n xraylab-1 kafkatopics
```
---