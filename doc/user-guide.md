# Appsody Operator

The Appsody Operator can be used to deploy applications created by [Appsody Application Stacks](https://appsody.dev/) into [OKD](https://www.okd.io/) or [OpenShift](https://www.openshift.com/) clusters.

## Operator installation

Use the instructions for one of the [releases](../deploy/releases) to install the operator into a Kubernetes cluster.

The Appsody Operator can be installed to:

- watch own namespace
- watch another namespace
- watch multiple namespaces
- watch all namespaces in the cluster

Appropriate cluster roles and bindings are required to watch another namespace, watch multiple namespaces or watch all namespaces.

## Overview

The architecture of the Appsody Operator follows the basic controller pattern:  the Operator container with the controller is deployed into a Pod and listens for incoming resources with `Kind: AppsodyApplication`. Creating an `AppsodyApplication` custom resource (CR) triggers the Appsody Operator to create, update or delete Kubernetes resources needed by the application to run on your cluster.

Each instance of `AppsodyApplication` CR represents the application to be deployed on the cluster:

```yaml
apiVersion: appsody.dev/v1beta1
kind: AppsodyApplication
metadata:
  name: my-appsody-app
spec:
  stack: java-microprofile
  applicationImage: quay.io/my-repo/my-app:1.0
  service:
    type: ClusterIP
    port: 9080
  expose: true
  storage:
    size: 2Gi
    mountPath: "/logs"
```

## Configuration

### Custom Resource Definition (CRD)

The following table lists configurable parameters of the `AppsodyApplication` CRD. For complete OpenAPI v3 representation of these values please see [`AppsodyApplication` CRD](../deploy/crds/appsody_v1beta1_appsodyapplication_crd.yaml).

Each `AppsodyApplication` CR must specify `applicationImage` parameter. Specifying other parameters is optional.

| Parameter | Description |
|---|---|
| `stack` | The name of the Appsody Application Stack that produced this application image. |
| `version` | The current version of the application. Label `app.kubernetes.io/version` will be added to all resources when the version is defined. |
| `serviceAccountName` | The name of the OpenShift service account to be used during deployment. |
| `applicationImage` | The absolute name of the image to be deployed, containing the registry and the tag. |
| `pullPolicy` | The policy used when pulling the image.  One of: `Always`, `Never`, and `IfNotPresent`. |
| `pullSecret` | If using a registry that requires authentication, the name of the secret containing credentials. |
| `architecture` | An array of architectures to be considered for deployment. Their position in the array indicates preference. |
| `service.port` | The port exposed by the container. |
| `service.type` | The Kubernetes [Service Type](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types). |
| `service.annotations` | Annotations to be added to the service. |
| `createKnativeService`   | A boolean to toggle the creation of Knative resources and usage of Knative serving. |
| `expose`   | A boolean that toggles the external exposure of this deployment via a Route or a Knative Route resource.|
| `replicas` | The static number of desired replica pods that run simultaneously. |
| `autoscaling.maxReplicas` | Required field for autoscaling. Upper limit for the number of pods that can be set by the autoscaler. It cannot be lower than the minimum number of replicas. |
| `autoscaling.minReplicas`   | Lower limit for the number of pods that can be set by the autoscaler. |
| `autoscaling.targetCPUUtilizationPercentage`   | Target average CPU utilization (represented as a percentage of requested CPU) over all the pods. |
| `resourceConstraints.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). Required field for autoscaling. |
| `resourceConstraints.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki.|
| `resourceConstraints.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). |
| `resourceConstraints.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki.|
| `env`   | An array of environment variables following the format of `{name, value}`, where value is a simple string. |
| `envFrom`   | An array of environment variables following the format of `{name, valueFrom}`, where `valueFrom` is YAML object containing a property named either `secretKeyRef` or `configMapKeyRef`, which in turn contain the properties `name` and `key`.|
| `readinessProbe`   | A YAML object configuring the [Kubernetes readiness probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/#define-readiness-probes) that controls when the pod is ready to receive traffic. |
| `livenessProbe` | A YAML object configuring the [Kubernetes liveness probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/#define-a-liveness-http-request) that controls when Kubernetes needs to restart the pod.|
| `volumes` | A YAML object representing a [pod volume](https://kubernetes.io/docs/concepts/storage/volumes). |
| `volumeMounts` | A YAML object representing a [pod volumeMount](https://kubernetes.io/docs/concepts/storage/volumes/). |
| `storage.size` | A convenient field to set the size of the persisted storage. Can be overridden by the `storage.volumeClaimTemplate` property. |
| `storage.mountPath` | The directory inside the container where this persisted storage will be bound to. |
| `storage.volumeClaimTemplate` | A YAML object representing a [volumeClaimTemplate](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#components) component of a `StatefulSet`. |
| `monitoring.labels` | Labels to set on [ServiceMonitor](https://github.com/coreos/prometheus-operator/blob/master/Documentation/api.md#servicemonitor). |
| `monitoring.endpoints` | A YAML snippet representing an array of [Endpoint](https://github.com/coreos/prometheus-operator/blob/master/Documentation/api.md#endpoint) component from ServiceMonitor. |
| `createAppDefinition`   | A boolean to toggle the automatic configuration of `AppsodyApplication`'s Kubernetes resources to allow creation of an application definition by [kAppNav](https://kappnav.io/). The default value is `true`. See [Application Navigator](#kubernetes-application-navigator-kappnav-support) for more information. |

### Basic usage

To deploy a Docker image containing an Appsody based application to a Kubernetes environment you can use the following CR:

 ```yaml
apiVersion: appsody.dev/v1beta1
kind: AppsodyApplication
metadata:
  name: my-appsody-app
spec:
  stack: java-microprofile
  applicationImage: quay.io/my-repo/my-app:1.0
```

The `applicationImage` value is required to be defined in `AppsodyApplication` CR. The `stack` should be the same value as the [Appsody application stack](https://github.com/appsody/stacks) you used to create your application.

### Service account

The operator can create a `ServiceAccount` resource when deploying an Appsody based application. If `serviceAccountName` is not specified in a CR, the operator creates a service account with the same name as the CR (e.g. `my-appsody-app`).

Users can also specify `serviceAccountName` when they want to create a service account manually.

If applications require specific permissions but still want the operator to create a `ServiceAccount`, users can still manually create a role binding to bind a role to the service account created by the operator. To learn more about Role-based access control (RBAC), see Kubernetes [documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).

### Labels

By default, the operator adds the following labels into all resources created for an `AppsodyApplication` CR: `app.kubernetes.io/name`, `app.kubernetes.io/managed-by`, `app.appsody.dev/stack` and `app.kubernetes.io/version` (only when `version` is defined). You can set new labels in addition to the pre-existing ones or overwrite them, excluding the `app.kubernetes.io/name` label. To set labels, specify them in your CR as key/value pairs.

```yaml
apiVersion: appsody.dev/v1beta1
kind: AppsodyApplication
metadata:
  name: my-appsody-app
  labels:
    my-label-key: my-label-value
spec:
  stack: java-microprofile
  applicationImage: quay.io/my-repo/my-app:1.0
```

_After the initial deployment of `AppsodyApplication`, any changes to its labels would be applied only when one of the parameters from `spec` is updated._

### Environment variables

You can set environment variables for your application container. To set environment variables, specify `env` and/or `envFrom` fields in your CR. The environment variables can come directly from key/value pairs, `ConfigMap`s or `Secret`s.

 ```yaml
apiVersion: appsody.dev/v1beta1
kind: AppsodyApplication
metadata:
  name: my-appsody-app
spec:
  stack: java-microprofile
  applicationImage: quay.io/my-repo/my-app:1.0
  env:
    - name: DB_PORT
      value: "6379"
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credential
          key: adminUsername
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credential
          key: adminPassword
  envFrom:
    - configMapRef:
        name: env-configmap
    - secretRef:
        name: env-secrets
```

Use `envFrom` to define all data in a `ConfigMap` or a `Secret` as environment variables in a container. Keys from `ConfigMap` or `Secret` resources become environment variable name in your container.

### High availability

Run multiple instances of your application for high availability using one of the following mechanisms: 
 - specify a static number of instances to run at all times using `replicas` parameter
 
    _OR_

 - configure auto-scaling to create (and delete) instances based on resource consumption using the `autoscaling` parameter.
      - Parameters `autoscaling.maxReplicas` and `resourceConstraints.requests.cpu` MUST be specified for auto-scaling.

### Persistence

Appsody Operator is capable of creating a `StatefulSet` and `PersistentVolumeClaim` for each pod if storage is specified in the `AppsodyApplication` CR.

Users also can provide mount points for their application. There are 2 ways to enable storage.

#### Basic storage

With the `AppsodyApplication` CR definition below the operator will create `PersistentVolumeClaim` called `pvc` with the size of `1Gi` and `ReadWriteOnce` access mode.

The operator will also create a volume mount for the `StatefulSet` mounting to `/data` folder. You can use `volumeMounts` field instead of `storage.mountPath` if you require to persist more then one folder.

```yaml
apiVersion: appsody.dev/v1beta1
kind: AppsodyApplication
metadata:
  name: my-appsody-app
spec:
  stack: java-microprofile
  applicationImage: quay.io/my-repo/my-app:1.0
  storage:
    size: 1Gi
    mountPath: "/data"
```

#### Advanced storage

Appsody Operator allows users to provide entire `volumeClaimTemplate` for full control over automatically created `PersistentVolumeClaim`.

It is also possible to create multiple volume mount points for persistent volume using `volumeMounts` field as shown below. You can still use `storage.mountPath` if you require only a single mount point.

```yaml
apiVersion: appsody.dev/v1beta1
kind: AppsodyApplication
metadata:
  name: my-appsody-app
spec:
  stack: java-microprofile
  applicationImage: quay.io/my-repo/my-app:1.0
  volumeMounts:
  - name: pvc
    mountPath: /data_1
    subPath: data_1
  - name: pvc
    mountPath: /data_2
    subPath: data_2
  storage:
    volumeClaimTemplate:
      metadata:
        name: pvc
      spec:
        accessModes:
        - "ReadWriteMany"
        storageClassName: 'glusterfs'
        resources:
          requests:
            storage: 1Gi
```
### Monitoring

Appsody Operator can create a `ServiceMonitor` resource to integrate with `Prometheus Operator`.

_This feature does not support integration with Knative Service. Prometheus Operator is required to use ServiceMonitor._

#### Basic monitoring specification

At minimum, a label needs to be provided that Prometheus expects to be set on `ServiceMonitor` objects. In this case, it is `apps-prometheus`.

```yaml
apiVersion: appsody.dev/v1beta1
kind: AppsodyApplication
metadata:
  name: my-appsody-app
spec:
  stack: java-microprofile
  applicationImage: quay.io/my-repo/my-app:1.0
  monitoring:
    labels:
       apps-prometheus: ''
```

#### Advanced monitoring specification

For advanced scenarios, it is possible to set many `ServicerMonitor` settings such as authentication secret using [Prometheus Endpoint](https://github.com/coreos/prometheus-operator/blob/master/Documentation/api.md#endpoint)

```yaml
apiVersion: appsody.dev/v1beta1
kind: AppsodyApplication
metadata:
  name: my-appsody-app
spec:
  stack: java-microprofile
  applicationImage: quay.io/my-repo/my-app:1.0
  monitoring:
    labels:
       app-prometheus: ''
    endpoints:
    - interval: '30s'
      basicAuth:
        username:
          key: username
          name: metrics-secret
        password:
          key: password
          name: metrics-secret
      tlsConfig:
        insecureSkipVerify: true
```

### Knative support

Appsody Operator can deploy serverless applications with [Knative](https://knative.dev/docs/) on a Kubernetes cluster. To achieve this, the operator creates a [Knative `Service`](https://github.com/knative/serving/blob/master/docs/spec/spec.md#service) resource which manages the whole life cycle of a workload.

To create Knative service, set `createKnativeService` to `true`:

```yaml
apiVersion: appsody.dev/v1beta1
kind: AppsodyApplication
metadata:
  name: my-appsody-app
spec:
  stack: java-microprofile
  applicationImage: quay.io/my-repo/my-app:1.0
  createKnativeService: true
```

By setting this parameter, the operator creates a Knative service in the cluster and populates the resource with applicable `AppsodyApplication` fields. Also, it ensures non-Knative resources including Kubernetes `Service`, `Route`, `Deployment` and etc. are deleted.

The CRD fields which are used to populate the Knative service resource include `applicationImage`, `serviceAccountName`, `livenessProbe`, `readinessProbe`, `service.Port`, `volumes`, `volumeMounts`, `env`, `envFrom`, `pullSecret` and `pullPolicy`.

For more details on how to configure Knative for tasks such as enabling HTTPS connections and setting up a custom domain, checkout [Knative Documentation](https://knative.dev/docs/serving/).

_Autoscaling related fields in `AppsodyApplication` are not used to configure Knative Pod Autoscaler (KPA). To learn more about how to configure KPA, see [Configuring the Autoscaler](https://knative.dev/docs/serving/configuring-the-autoscaler/)._

_This feature is only available if you have Knative installed on your cluster._

### Exposing service externally

#### Non-Knative deployment

To expose your application externally, set `expose` to `true`:

```yaml
apiVersion: appsody.dev/v1beta1
kind: AppsodyApplication
metadata:
  name: my-appsody-app
spec:
  stack: java-microprofile
  applicationImage: quay.io/my-repo/my-app:1.0
  expose: true
```

By setting this parameter, the operator creates an unsecured route based on your application service. Setting this parameter is the same as running `oc expose service <service-name>`.

To create a secured HTTPS route, see [secured routes](https://docs.openshift.com/container-platform/3.11/architecture/networking/routes.html#secured-routes) for more information.

_This feature is only available if you are running on OKD or OpenShift._

##### Canary deployment using `Route`

You can easily test a new version of your application using the Canary deployment methodology by levering the traffic split capability built into OKD's `Route` resource.
*  deploy the first version of the application using the instructions above with `expose: true`, which will create an OKD `Route`.
*  when a new application version is available, deploy it via the Appsody Operator but this time choose `expose: false`.
*  edit the first application's `Route` resource to split the traffic between the two services using the desired percentage.  

    Here is a screenshot of the split via the OKD UI:

    ![Traffic Split](route.png)

    Here is the corresponding YAML, which you can edit using the OKD UI or simply using `oc get route <routeID>` and then `oc apply -f <routeYAML>`:

    ```yaml
    apiVersion: route.openshift.io/v1
    kind: Route
    metadata:
      labels:
        app: my-appsody-app-1
      name: canary-route
    spec:
      alternateBackends:
        - kind: Service
          name: my-appsody-app-2
          weight: 20
      host: canary-route-testing.my-host.com
      port:
        targetPort: 9080-tcp
      to:
        kind: Service
        name: my-appsody-app-1
        weight: 80
    ```      

*  once you are satisfied with the results you can simply route 100% of the traffic by switching the `Route`'s `spec.to` object to point to `my-appsody-app-2` at a weight of 100 and remove the `spec.alternateBackends` object. This can similarly be done via the OKD UI.

#### Knative deployment

To expose your application as a Knative service externally, set `expose` to `true`:

```yaml
apiVersion: appsody.dev/v1beta1
kind: AppsodyApplication
metadata:
  name: my-appsody-app
spec:
  stack: java-microprofile
  applicationImage: quay.io/my-repo/my-app:1.0
  createKnativeService: true
  expose: true
```

When `expose` is **not** set to `true`, the Knative service is labeled with `serving.knative.dev/visibility=cluster-local` which makes the Knative route to only be available on the cluster-local network (and not on the public Internet). However, if `expose` is set `true`, the Knative route would be accessible externally.

To configure secure HTTPS connections for your deployment, see [Configuring HTTPS with TLS certificates](https://knative.dev/docs/serving/using-a-tls-cert/) for more information.

### Operator Configuration

When the operator starts, it creates two `ConfigMap` objects that contain default and constant values for individual stacks in `AppsodyApplication`.

#### Stack defaults

ConfigMap [`appsody-operator-defaults`](../deploy/stack_defaults.yaml) contains the default values for each stack. When users do not provide values inside their `AppsodyApplication` resource, the operator will look up default values inside
this [stack defaults map](../deploy/stack_defaults.yaml).

Input resource:

```yaml
apiVersion: appsody.dev/v1beta1
kind: AppsodyApplication
metadata:
  name: my-appsody-app
spec:
  stack: java-microprofile
  applicationImage: quay.io/my-repo/my-app:1.0
```

Since in the `AppsodyApplication` resource service `port` and `type` are not set, they will be looked up in the default `ConfigMap` and added to the resource. It will be set according to the `stack` field. If the `appsody-operator-defaults` doesn't have the `stack` with a particular name defined then the operator will use `generic` stack's default values.

After defaults are applied:

```yaml
apiVersion: appsody.dev/v1beta1
kind: AppsodyApplication
metadata:
  name: my-appsody-app
spec:
  stack: java-microprofile
  applicationImage: quay.io/my-repo/my-app:1.0
  ....
  service:
    port: 9080
    type: ClusterIP  
```
 
#### Stack Constants ConfigMap

[`appsody-operator-constants`](../deploy/stack_constants.yaml) ConfigMap contains the constant values for each stack. These values will always be used over the ones that users provide. This can be used to limit user's ability to control certain fields such as `expose`. It also provides the ability to set environment variables that are always required.

Input resource:

```yaml
apiVersion: appsody.dev/v1beta1
kind: AppsodyApplication
metadata:
  name: my-appsody-app
spec:
  stack: java-microprofile
  applicationImage: quay.io/my-repo/my-app:1.0
  expose: true
  env:
  -  name: DB_URL
     value: url
```

After constants are applied:

```yaml
apiVersion: appsody.dev/v1beta1
kind: AppsodyApplication
metadata:
  name: my-appsody-app
spec:
  stack: java-microprofile
  applicationImage: quay.io/my-repo/my-app:1.0
  ....
  expose: false
  env:
  -  name: VENDOR
     value: COMPANY
  -  name: DB_URL
     value: url     
```

### Kubernetes Application Navigator (kAppNav) support

By default, Appsody Operator configures the Kubernetes resources it generates to allow automatic creation of an application definition by [kAppNav](https://kappnav.io/), Kubernetes Application Navigator. You can easily view and manage the deployed resources that comprise your application using Application Navigator. You can disable auto-creation by setting `createAppDefinition` to `false`.

To join an existing application definition, disable auto-creation and set the label(s) needed to join the application on `AppsodyApplication` CR. See [Labels](#labels) section for more information.

_This feature is only available if you have kAppNav installed on your cluster. Auto creation of an application definition is not supported when Knative service is created_

### Troubleshooting

See the [troubleshooting guide](troubleshooting.md) for information on how to investigate and resolve deployment problems.
