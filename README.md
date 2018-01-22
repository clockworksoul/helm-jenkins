# Jenkins Helm Chart

A fully-customizable Jenkins master+agent cluster utilizing the Jenkins Kubernetes plugin.

* https://wiki.jenkins-ci.org/display/JENKINS/Kubernetes+Plugin

Forked and greatly modified from the official Kubernetes "stable" chart:

* https://github.com/kubernetes/charts/tree/master/stable/jenkins

Inspired by the awesome work of Carlos Sanchez <carlos@apache.org>.

## Chart Details
This chart will do the following:

* Create 1 x Jenkins master instance
* Define master persistent state (if any) stored in a [Kubernetes PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
* Define one or more custom worker agents
* Define a default admin user (credentials random by default)
* Install default plugins
* Install default GitHub keys from an existing Secret resource
* Install zero to N shared libraries
* Expose (internally) a metrics endpoint (`/metricz` by default) using the [Prometheus plugin](https://wiki.jenkins.io/display/JENKINS/Prometheus+Plugin)
* Create _either_:
  1. An Ingress resource supporting the specified hostname and protocol (http/https)
  1. An external LoadBalancer exposing port 8080

## Installing the Chart

First, you'll need a Kubernetes cluster of version 1.7 or higher, and [Helm](https://github.com/kubernetes/helm) utility, installed and configured.

> **Tip**: The [kops-in-a-can](https://github.com/clockworksoul/kops-in-a-can) tool can greatly simplify the installation process.

To install the chart with the release name `my-release`:

```bash
$ helm install --name my-release jenkins
```

## Configuration

The following tables lists the configurable parameters of the Jenkins chart and their default values.

### Jenkins Master

| Parameter                         | Description                          | Default                                                                      |
| --------------------------------- | ------------------------------------ | ---------------------------------------------------------------------------- |
| `master.name`                     | Jenkins master name                  | `jenkins-master`                                                             |
| `master.image.repository`         | Master image name                    | `jenkinsci/jenkins`                                                          |
| `master.image.tag`                | Master image tag                     | `2.102`                                                                      |
| `master.image.pullPolicy`         | Master image pull policy             | `Always`                                                                     |
| `master.image.pullSecret`         | Master image pull secret             | Not set                                                                      |
| `master.component`                | k8s selector key                     | `jenkins-master`                                                             |
| `master.useSecurity`              | Use basic security                   | `true`                                                                       |
| `master.adminUser`                | Admin username (and password) created as a secret if useSecurity is true | `admin`                                  |
| `master.resources`                | Master [resource requests](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/) block | `requests.memory: 512m requests.cpu: 200m` |
| `master.javaOpts`                 | Master JVM options.                  | `-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -Djava.awt.headless=true` |
| `master.jenkinUrl`                | Master external Jenkins URL          | `http://ci.temporary.com`                                                    |
| `master.jenkinsInitClobbersConfig`| Master configs overwritten on init   | `false`                                                                      |
| `master.serviceAnnotations`       | Service annotations                  | `{}`                                                                         |
| `master.secretKeyName`            | Master container SSH keys (mounted to ~/.ssh) | Not set                                                             |
| `master.serviceType`              | k8s service type                     | `LoadBalancer`                                                                  |
| `master.servicePort`              | k8s service port                     | `8080`                                                                       |
| `master.nodePort`                 | k8s node port                        | Not set                                                                      |
| `master.containerPort`            | Master listening port                | `8080`                                                                       |
| `master.slaveListenerPort`        | Listening port for agents            | `50000`                                                                      |
| `master.loadBalancerSourceRanges` | Allowed inbound IP addresses         | `0.0.0.0/0`                                                                  |
| `master.loadBalancerIP`           | Optional fixed external IP           | Not set                                                                      |
| `master.jmxPort`                  | Open a port, for JMX stats           | Not set                                                                      |
| `master.customConfigMap`          | Use a custom ConfigMap               | `false`                                                                      |
| `master.initScripts`              | List of Jenkins init scripts         | Not set                                                                      |
| `master.installPlugins`           | List of Jenkins plugins to install   | `credentials-binding:1.13 docker-build-publish:1.3.2 git:3.7.0 kubernetes:1.1.2 workflow-aggregator:2.5 workflow-job:2.16` |
| `master.scriptApproval`           | List of groovy functions to approve  | Not set                                                                      |
| `master.nodeSelector`             | Node labels for pod assignment       | `{}`                                                                         |
| `master.tolerations`              | Toleration labels for pod assignment | `{}`                                                                         |

### Jenkins Agents

The base helm chart has been extended to add support for multiple Jenkins agents, each with its own labels and other characteristics. 

The following example shows two defined agents: one for Golang builds, and another for Maven builds.

```yaml
agents:
  - component: "jenkins-agent-go"
    enabled: true
    image:
      repository: "your-repo/jenkins-agent-go"
      tag: "3.16-1"
      pullPolicy: "Always"
    labels: "go golang"

  - component: "jenkins-agent-maven"
    enabled: true
    image:
      repository: "your-repo/jenkins-agent-maven"
      tag: "3.16-1"
      pullPolicy: "Always"
    labels: "maven"
```

| Parameter                 | Description                                     | Default                |
| ------------------------- | ----------------------------------------------- | ---------------------- |
| `agents.component`        | The agent name                                  | `default`              |
| `agents.enabled`          | Enable Kubernetes plugin jnlp-agent podTemplate | `true`                 |
| `agents.privileged`       | Agent privileged container                      | `false`                |
| `agents.image.repository` | Agent image repository                          | `jenkins/jnlp-slave`   |
| `agents.image.tag`        | Agent image repository                          | `3.16-1`               |
| `agents.image.pullPolicy` | Always pull agent container image before build  | `false`                |
| `agents.image.pullSecret` | Agent image pull secret                         | Not set                |
| `agents.labels`           | 1 to N space-delimitted labels                  | `base default`         |
| `agents.resources`        | Agent [resource requests](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/) block | `requests.memory: 256Mi requests.cpu: 200m` |
| `agents.javaOpts`         | Agent JVM options                               | `-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap` |
| `agents.envVars`          | Agent injected environment variables            | Not set                |
| `agents.nodeSelector`     | Agent image tag                                 | `{}`                   |
| `agents.secretKeyName`    | Agent container SSH keys (mounted to `~/.ssh`)  | Not set                |
| `agents.volumes`          | Additional volumes                              | `nil`                  |

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`.

Alternatively, a YAML file that specifies the values for the parameters can be provided while installing the chart. For example,

```bash
$ helm install --name my-release -f values.yaml jenkins
```

> **Tip**: You can use the default [values.yaml](values.yaml)

#### Mounting volumes into your Agent pods

Your Jenkins Agents will run as pods, and it's possible to inject volumes where needed:

```yaml
agent:
  - component: my-agent
    volumes:
    - type: Secret
      secretName: jenkins-mysecrets
      mountPath: /var/run/secrets/jenkins-mysecrets
```

The suported volume types are: `ConfigMap`, `EmptyDir`, `HostPath`, `Nfs`, `Pod`, `Secret`. Each type supports a different set of configurable attributes, defined by [the corresponding Java class](https://github.com/jenkinsci/kubernetes-plugin/tree/master/src/main/java/org/csanchez/jenkins/plugins/kubernetes/volumes).

> **Important**: As a bonus the following volumes are automatically created for each volume:

```yaml
volumes:
  - type: HostPath
    hostPath: /var/run/docker.sock
    mountPath: /var/run/docker.sock
  - type: HostPath
    hostPath: /var/jenkins_home/workspace
    mountPath: /var/jenkins_home/workspace
```

## Ingress

If the value of `ingress.host` is defined, a [Kubernetes Ingress resource](https://kubernetes.io/docs/concepts/services-networking/ingress/) will be created associating that forwards traffic addressed to the specified host name to the Jenkins master service.

| Parameter                         | Description                          | Default                                                                      |
| --------------------------------- | ------------------------------------ | ---------------------------------------------------------------------------- |
| `ingress.host`                    | Ingress host address                 | `ci.temporary.com`                                                           |
| `ingress.annotations`             | Ingress annotations                  | `{}`                                                                         |
| `ingress.tls`                     | Ingress TLS configuration            | `[]`                                                                         |

## NetworkPolicy

| Parameter                         | Description                                 | Default       |
| --------------------------------- | ------------------------------------------- | --------------|
| `networkPolicy.enabled`           | Enable creation of NetworkPolicy resources. | `false`       |

To make use of the NetworkPolicy resources created by default,
install [a networking plugin that implements the Kubernetes
NetworkPolicy spec](https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy#before-you-begin).

For Kubernetes v1.5 & v1.6, you must also turn on NetworkPolicy by setting
the DefaultDeny namespace annotation. Note: this will enforce policy for _all_ pods in the namespace:

    kubectl annotate namespace default "net.beta.kubernetes.io/network-policy={\"ingress\":{\"isolation\":\"DefaultDeny\"}}"


Install helm chart with network policy enabled: 

    $ helm install jenkins --set networkPolicy.enabled=true

## Persistence

The Jenkins image stores persistence under `/var/jenkins_home` path of the container. A dynamically managed Persistent Volume
Claim is used to keep the data across deployments, by default. This is known to work in GCE, AWS, and minikube. Alternatively,
a previously configured Persistent Volume Claim can be used.

It is possible to mount several volumes using `persistence.volumes` and `persistence.mounts` parameters.

### Persistence Values

| Parameter                   | Description                     | Default         |
| --------------------------- | ------------------------------- | --------------- |
| `persistence.enabled`       | Enable the use of a Jenkins PVC | `true`          |
| `persistence.existingClaim` | Provide the name of a PVC       | Not set         |
| `persistence.storageClass`  | Provide the storage class of a PVC | Not set      |
| `persistence.accessMode`    | The PVC access mode             | `ReadWriteOnce` |
| `persistence.annotations`   | Enable the use of a Jenkins PVC | `{}`            |
| `persistence.size`          | The size of the PVC             | `8Gi`           |
| `persistence.volumes`       | Additional volumes              | Not set         |
| `persistence.mounts`        | Additional mounts               | Not set         |


#### Existing PersistentVolumeClaim

1. Create the PersistentVolume
1. Create the PersistentVolumeClaim
1. Install the chart
```bash
$ helm install --name my-release --set persistence.existingClaim=PVC_NAME jenkins
```

## Custom ConfigMap

When creating a new chart with this chart as a dependency, CustomConfigMap can be used to override the default config.xml provided.
It also allows for providing additional xml configuration files that will be copied into `/var/jenkins_home`. In the parent chart's values.yaml,
set the value to true and provide the file `templates/config.yaml` for your use case. If you start by copying `config.yaml` from this chart and
want to access values from this chart you must change all references from `.Values` to `.Values.jenkins`.

```
jenkins:
  master:
    customConfigMap: true
```

## RBAC

| Parameter                         | Description                          | Default                                   |
| --------------------------------- | ------------------------------------ | ----------------------------------------- |
| `rbac.install`                    | Create service account and ClusterRoleBinding for Kubernetes plugin | `false`    |
| `rbac.serviceAccountName`         | RBAC ServiceAccount name             | `default`                                 |
| `rbac.roleRef`                    | Cluster role name to bind to         | `cluster-admin`                           |

If running upon a cluster with RBAC enabled you will need to do the following:

* `helm install jenkins --set rbac.install=true`
* Create a Jenkins credential of type Kubernetes service account with service account name provided in the `helm status` output.
* Under configure Jenkins -- Update the credentials config in the cloud section to use the service account credential you created in the step above.
