---
title: "Deploying a Remote Engine on Kubernetes (Helm)"
slug: "remote-engine-deployment-kubernetes"
updated: 2026-07-15T11:30:07Z
published: 2026-07-15T11:30:07Z
canonical: "docs.nonamesecurity.com/remote-engine-deployment-kubernetes"
---

> ## Documentation Index
> Fetch the complete documentation index at: https://docs.nonamesecurity.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Deploying a Remote Engine on Kubernetes (Helm)

## Overview

For more information about remote engines, see the [Remote Engine Overview](/v368/docs/multiple-engine-datasheet).

This page describes the prerequisites, configuration, and steps to deploy a Remote engine on Kubernetes using Helm.

## Deployment Considerations

- For best practices of deploying on Kubernetes, see the [best practices](/v368/docs/remote-engine-deployment-kubernetes-best-practices) doc.
- A remote engine with a single worker has a specific limit based on the node sizing. See [sizing requirements](/v368/docs/multiple-engine-datasheet#engine-sizing) for details. If your expected traffic load exceeds this limit, we suggest using multiple remote engines.

Customization Support

Please note that any customizations made by the customer to the product or service are not guaranteed or warrantied by our product, service, or support teams. While we may provide guidance or assistance in implementing these customizations, they are undertaken at the customer's own risk. Our support is focused on ensuring the functionality and stability of the standard product offering, and any issues arising from customer customizations may fall outside the scope of our support agreements.

### Pod Security Standards

Remote Engine deployments on Kubernetes default to the Pod Security Standards (PSS) **baseline** profile and are compatible with **privileged** environments. If your cluster enforces the **restricted** profile, you may need to override the default Kubernetes `securityContext` settings to meet the stricter requirements. Refer to the [Kubernetes Pod Security Standards documentation](https://kubernetes.io/docs/concepts/security/pod-security-standards/) for guidance. You can apply overrides per chart or per service.

Note: Security Context Overrides Replace Defaults

Setting `global.securityContext` completely replaces the default configuration. Kubernetes doesn’t merge security context values. You must define all required settings in the override.

Traffic Mirroring Support

Traffic mirroring (NIC Integration) on Kubernetes-based engines is supported only when the engine runs with the **privileged** Pod Security Standards profile.

### Multiple Workers

For a single Kubernetes deployment, you can create multiple engine workers. Multiple workers enable you to horizontally scale your engine capacity, for better performance and load balancing. You configure traffic routing rules to each worker based on the traffic's destination host or path.

We recommend using multiple workers if:

- you exceed the maximum capacity for your node hardware and cannot scale vertically.
- you have an traffic source that produces more than the maximum capacity of traffic for your node hardware.

Note

- You can set up to a maximum of 10 workers per remote engine deployment (soft limitation)
- You cannot change the number of workers once a remote engine is configrued. If you need to change the number of workers, you must redeploy the engine.
- You can change the traffic routing at any time (without redeploying).

## Requirements

- See [Prerequisites](/docs/remote-engine-deployment-kubernetes#prerequisites)
- Administrator privileges are required to:
  - Create namespace
  - Create roles
  - Create secrets
  - Create persistent volumes and persistent volume claims
  - Create deployments and stateful sets
  - Create Ingress (optional)
  - Create a load balancer (optional)

### Prerequisites

- AMD or ARM processor.
- The Kubernetes cluster: Kubernetes api version 1.18+
- Permission to create a persistent volume claim, or an already provisioned persistent volume claim. The volume claim is required by the Engine. We recommend the size of the persistent volume claim be 30 GB (per engine), but it should be no less than 15 GB.
- Kubectl utility, configured to communicate to the Kubernetes cluster. For more information, see the [Kubernetes Install Tools](https://kubernetes.io/docs/tasks/tools/) documentation.
- Helm utility, version 3.9. For more information, see the [Helm installation documentation](https://helm.sh/docs/intro/install/), -
- Ingress Controller installed in your Kubernetes cluster (for example NGINX Ingress Controller or ALB Ingress Controller). For more information about Kubernetes Ingress and Ingress Controller, see this [Kubernetes Ingress practical guide](https://www.solo.io/topics/kubernetes-api-gateway/kubernetes-ingress/).
Note

By default, Kubernetes Ingress does not support TCP or UDP services. If you want to integrate sources with TCP/UDP traffic, enable TCP/UDP services. See for example [NGINX Ingress](https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/).
- Metrics Server or an equivalent solution installed. Note that to use Horizontal Pod Autoscaler (HPA), you must install Metrics Server on a cluster where HPA is supposed to work. Learn [how to install Metrics Server](https://github.com/kubernetes-sigs/metrics-server).
- A FQDN for the remote engine, and administrator rights for DNS to create a CNAME record for this FQDN. This FQDN server record will point to the Ingress created by Helm.
- The following images are delivered with the download package. If you want to use your own docker registry, the helm custom values yaml file that Akamai provides during installation contains credentials to access the Akamai docker registry. Pull these images from the Akamai registry and place them in yours.
  - cloud-metadata:VERSION - Pulls metadata from a cloud environment.
  - light-engine:VERSION - An auto-scaling engine component performing simpler analysis tasks, dynamically adjusting to traffic loads
  - heavy-engine:VERSION - An engine component aggregating multiple light services to perform more comprehensive analyses on a 30-second cycle
  - integrations-adapter:VERSION - Prevention and fetch configuration
  - nats-jetstream:2.12.3-custom - Event service
  - nginx:VERSION - NGINX
  - nginx:VERSION-alpine - A version of NGINX based on Alpine Linux
  - nogate:VERSION - Forward proxy
  - router:VERSION - Queues incoming traffic for engine.
  - platform-integrations:VERSION - Fetches and processes traffic from logs. Required only if you install access log-based integrations.
  - postgres:15.10-alpine (not required when using an external Postgres Database) - DB, required only if you use platform.
  - syslog-ng:linuxserver-4.1.1 - Engine logging

Before deployment, ensure that you meet the following conditions:

- The version of the remote engine is identical to the version of the API Security Server it will communicate with.
- The environment server (cloud or other) has the following ports open for incoming traffic:
  

| Port | Protocol | Description/Operations |
| --- | --- | --- |
| 80 | TCP | Traffic source integration data (API packet pairs) if sent via HTTP |
| 443 | TCP | Traffic source integration data (API packet pairs) if sent via HTTPS. For traffic mirroring based on a plugin installed on an external API management system. |
- The environment server (cloud or other) has the following ports open for outbound traffic:
  - The remote engine sends API metadata to the API Security Server on the server's port 443.
  - You must open an outbound port TCP 443 to the following servers:
    - **<your_organization>.nonamesec.com** – For sending data to the API Security Server
    - **<your_organization>-mtls.nonamesec.com** – For sending data to the API Security Server using mTLS. If this endpoint is unavailable, the Remote Engine automatically falls back to TLS using **`&lt;your_organization&gt;.nonamesec.com`**.
    - **pkg.nonamesec.com** – For installing updates
    - **jfrog.cicd.nonamesec.com** or **us-central1-docker.pkg.dev** – For installing updates

          Securing Traffic to the Remote Engine

          

To make sure that the traffic is securely transmitted to your remote engine using TLS, place the engine behind a load balancer with a TLS certificate issued by a trusted Certificate Authority and use an HTTPS URL for your remote engine.

## Deploying a Remote Engine (Kubernetes / Helm)

To deploy remote engine on Kubernetes using Helm, follow the below steps:

[Step 1:](/docs/remote-engine-deployment-kubernetes#step-1-define-remote-engine) Define Remote Engine [Step 2:](/docs/remote-engine-deployment-kubernetes#step-2-configure-workers-settings-and-define-routing-rules) Configure workers settings and define routing rules [Step 3:](/docs/remote-engine-deployment-kubernetes#step-3-deploy-the-remote-engine) Deploy the Remote Engine

### Step 1: Define Remote Engine

1. On the [Engines page](/v368/docs/managing-remote-engines), select **Add Engine**.
2. Select **Kubernetes** and select **Next**.
3. For **Installation Method**, select **Helm**.
4. Enter a name for the remote engine and the URL to which integrations should send API traffic. The URL must include the protocol (http/https), and must end with `/engine`.
5. (Optional) Select **Advance Settings** and configure the following:
  1. Modify the URL or IP address of the API Security Server, if required.
  2. Select **Verify SSL certificate** to have the engine validate the server certificate.
  3. Select **HTTP proxy** to configure HTTP forward proxy for the communication between the engine and the API Security Server, and then:

Note

The platform automatically includes your forward proxy settings to the `values.yaml` file used during the installation. After you deploy the engine, the only way to modify the forward proxy configuration is to perform a new deployment.
    1. Enter the proxy **Host** (URL or IP address) and **Port**.
    2. (Optional) Configure forward proxy authentication:
    3. Select the **Basic Auth** option.
    4. Enter **Username** and **Password**.
6. If you require multiple workers, select **Multiple Workers** and continue with [Step 2](/docs/remote-engine-deployment-kubernetes#step-2-configure-workers-settings-and-define-routing-rules). Otherwise, select **Submit** to create the engine.
Note

After you save the engine, you cannot change a multiple worker engine to a single worker engine, or vice versa.

If you select **Multiple Workers**, additional fields appear on the page. Continue with the next step. Otherwise, select **Submit** to create the engine and continue with [Step 4](/docs/remote-engine-deployment-kubernetes#step-4-deploy-the-remote-engine).

### Step 2: Configure Workers Settings and Define Routing Rules

Follow this step only for multiple workers deployment

If you plan to use multiple workers, select the number of workers according the expected traffic load and define routing rules to direct traffic to specific workers based on target host/path. The goal is to direct approximately equal amounts of traffic to each worker.

![engine_settings_number_of_workers.png](https://cdn.document360.io/df10da0f-ae0c-4118-a445-f592c55794a2/Images/Documentation/engine_settings_number_of_workers.png)

1. Select the number of workers based on the expected maximum traffic RPS. As a guideline, use one worker for every 15k RPS (The actual number of RPS will depend on the schema and transaction size). You can select up to 10 workers (soft limitation). If you expect more than 150k RPS, contact Akamai support for assistance.  

**Note**: Once you save the worker configuration, you cannot change the number of workers.
2. Select whether to divide and route traffic according to the traffic's destination **Path** or **Host**. Select this value in a way that makes sense for your environment (because it enables you to direct approximately equal amount of traffic to each worker).
3. Create the routing rule for each worker.

Notes

  - You can add multiple patterns for each worker.
  - Patterns are PLAIN TEXT and are used to match the START of the destination path/host.
  - For example, a starting pattern for a path might be */example/v1*. A starting pattern for a host might be *example.com:8080*.
  - Traffic is routed according to the first matching pattern in the list.
  - All traffic that does not match any rule is routed to the default worker, which is Worker 1.
  - You can change the routing pattern later, as required.
  1. Select the worker.
  2. Enter a pattern to match the START of the traffic destination path/host.
  3. Select **Add Rule**.
  1. Select **Submit**. The engine is created.

### Step 3: Deploy the Remote Engine

After creating the Remote Engine, deploy it in your Kubernetes cluster using the commands provided in the UI.

Choose how to source container images:

- **Akamai Registry (recommended)** – use Akamai-hosted images
- **Private Registry** – push images to your private registry first

#### Option 1: Akamai Registry

1. Log in to the Helm registry using the command provided in the UI.
2. Download the generated `custom_values.yaml` file.

#### Option 2: Private Registry

1. Log in to the Helm registry using the command provided in the UI.
2. Load images to your registry using the script provided in the UI.
3. Download and update `custom_values.yaml` with your registry details.

#### Complete the Deployment

1. If needed, edit the `custom_values.yaml` file as described in [Appendix A - Helm Configuration](/docs/remote-engine-deployment-kubernetes#appendix-a-helm-configuration).

Define Your Ingress Data

To use an Ingress controller, include its required parameters in the `custom_values.yaml` file.

Use External Secret Management System

To store your secrets in an external secrets management system, update the configuration before deployment as described [here](/docs/remote-engine-deployment-kubernetes#using-external-secrets-management-system).
2. Install the Remote Engine using the command provided in the UI.
3. (Optional) Download the Helm chart for review.

The deployment is complete.

To verify the deployment, run the following command: `kubectl get ingress`. You should see something like:

```
NAME CLASS HOSTS ADDRESS PORTS AGE
YOUR_CLUSTER-ingress alb YOUR_FQDN_NONAME_HOST YOUR_REAL_LOAD_BALANCER_FQDN 80 15m
```

The engine should appear as active in the **Settings > Engines** page in the UI.

Registration Keys are Single-Use

The registration key is usable only for a single device. Once the remote engine registers with the API Security Server, it will not work on a another server so long as the engine continues to send traffic to the API Security Server. For more information, see [Secure Communication](/v368/docs/multiple-engine-datasheet#secure-communication).

## Updating a Remote Engine (Kubernetes / Helm)

Update On-Premises Platform

The API Security Server does not support remote engines with later versions than itself. So, for example, if the API Security Server is version 3.2, it does not support communication with remote engines of version 3.3.

The API Security Server supports remote engines with earlier versions. Nevertheless, remote engines with earlier versions may not support the new features of the most recent API Security Server version. If you have an on-premises deployment, we strongly recommend that you update all remote engines immediately after updating the API Security Server.

Also see [Engine Support By Version](/v368/docs/engine-support-by-version).

1. Download and unpack the new package and Helm chart from the [Engines page](/v368/docs/managing-remote-engines).
2. Move the Helm `custom_values.yaml` file to the deployment package in **kubernetes/nonamesec**.

Renewing mTLS Certificate

When renewing the mTLS certificate, you can copy the updated `re_client_certificate` and `re_client_private_key` values from the UI (**Settings** > **Engines** > **Install**) and update them in `custom_values.yaml` instead of replacing the entire file.

Use External Secrets Management System

To start using a custom external secrets management system for your engine, edit the configuration yaml files [following these guidelines](/docs/remote-engine-deployment-kubernetes#using-external-secrets-management-system).

3.. Continue with a deployment, as described in [Deployment](/docs/remote-engine-deployment-kubernetes#deploying-a-remote-engine-managed-kubernetes-helm).

Alternately, you can perform the following steps:

1. **Login**: Run `helm registry login us-central1-docker.pkg.dev/noname-artifacts/nns-docker/nonamesec -u _json_key_base64 -p &lt;token&gt;`. You can find the token in the Helm custom values Yaml file in **global.imageCredentials.password**.
2. **Download and Install**: In the same folder as your Helm `custom_values.yaml`, run `helm install nonamesec oci://us-central1-docker.pkg.dev/noname-artifacts/nns-docker/nonamesec -n &lt;namespace&gt; --version &lt;remote engine version&gt;`. Where version is, for example, *v3.26.1*. Make sure to include the *v*. You can use *v3.26.x* to get the latest version of 3.26. You can add the same parameters you used during installation.
  - **Note**: To download without running the installation, run `helm pull oci://us-central1-docker.pkg.dev/noname-artifacts/nns-docker/nonamesec --version &lt;remote engine version&gt;`.

## Uninstalling a Remote Engine (Kubernetes / Helm)

1. Find the deployment using `helm list --all-namespaces` and search for an entry with the chart name "noname".
2. Remove the application with `helm uninstall nonamesec -n noname`.
3. Delete the engine in the [Engines page](/v368/docs/managing-remote-engines).

## Appendix A: Helm Custom Values Yaml File

After you download the Helm custom values configuration Yaml file, verify these parameters and adjust them as required.

Advanced Configuration

Additional configurations exists in the `values.yaml`.

If your deployment requires additional modifications, see the [configuration reference](/v368/docs/remote-engine-deployment-kubernetes#configuration-reference) for Kubernetes.

### Using External Secrets Management System

When deploying your engine on a Kubernetes cluster, your secrets are stored, by default, in Kubernetes Secrets Manager. To use another external secrets management system of your choice rather than the default option, follow these steps:

1. Install External Secrets Operator following [these guidelines](https://external-secrets.io/latest/introduction/getting-started/).
2. From the `custom-values.yaml` file downloaded while creating your engine, copy three secrets: `engine_id`, `re_client_private_key`, and `re_client_certificate`, and place them in your secrets management system. Make sure the names of all secrets in your secrets management system match exactly with those in the file.
3. In the `custom-values.yam`l file replace the secrets you have just copied with any placeholder, for example:

```
engine:
    enabled: true
    engine_id: "12345678"
    re_client_certificate: "12345678"
    re_client_private_key: "12345678"
    remote_keys_encoded :"<SYSTEM_GENERATED_BASE64_VALUE>"
```

          Note

          

`remote_keys_encoded` is a system-generated, Base64-encoded value that already contains encrypted key material. Store it exactly as provided in your secrets management system.

1. Download the remote engine package.
2. Edit the `nonamesec/values.yaml` configuration file as follows:

```
nns_eso:
    enabled: true
    secret: "dev/customers/customer-test"
```
  1. Set `nns_eso.enabled` to true.
  2. Provide the path for your secrets management system by setting `nns_eso.secret` property.  

For example:
3. Edit `nonamesec/charts/nns-eso/values.yaml` file as follows:
  1. Change `secretStoreRef.name` value to your secrets management system’s name (defaults to `aws-secretsmanager` if not specified).
  2. (Optional) If needed, set `deploymentType` value. The possible values are:
    - `remote-engine` (default): for deploying a remote engine only
    - `full`: for performing a full API Security installation
    - `remote-platform`: for deploying an engine with platform components enabled
4. Continue with the deployment steps.

To revert to using the default secrets management system (Kubernetes Secrets Manager), set `nns_eso.enabled` to `false` in the `nonamesec/values.yaml` file and redeploy the engine.

### Configuration Reference

| Section | Parameter | Description | Default |
| --- | --- | --- | --- |
| global |  |  |  |
|  | `version` | API Security Server version | Generated by API Security |
|  | `readOnlyRootFileSystem` | When set to `true`, containers can't write to the internal filesystem. Writable directories must be defined per container under `extraVolumeMounts`. This parameter is not included in the UI-generated `custom-values.yaml` and must be added manually if modified. | `false` |
|  | `engine.env.MANAGEMENT_IP` | FQDN of the API Security Server | Generated by API Security |
|  | `engine.env.COLLECTOR_TYPE` | Set to `"true"` | Generated by API Security |
|  | `engine.env.DATA_ORCHESTRATION_ENABLE_SSL` | Communication type between services. Use `true`. | Generated by API Security |
|  | `engine.env.SETCAP` | Controls whether additional Linux capabilities required for NIC integration (for example NET_ADMIN, NET_RAW, SYS_NICE) are applied. Set to false for environments that do not allow capability escalation (for example AWS Fargate). | `true` |
|  | `engine.securityContext ` | (NIC integration / advanced networking only) Defines container security settings required for network interface access. Adds capabilities: SYS_NICE, NET_RAW, NET_ADMIN, SETFCAP, SETUID, SETGID (all other capabilities are dropped). | `enabled` |
|  | `engine.volume` | Engine persistent volume customization name | `empty` |
|  | `platform.enabled` | Whether to deploy the platform. Should be `true` in all cases except for remote engine without logs capability. | Generated by API Security |
|  | `router.autoscaling.enabled` |  | `false` |
|  | `router.replicas` | Set to the number of workers, for example `3` for three workers | `1` |
|  | `nats.replicas` | Use 0.5 per worker, rounded up to the nearest integer. For example, for 3 workers, use `2`. The minimum is 1. | Generated by API Security |
|  | `nginx.replicas` | Routes all internal app traffic. If traffic is too heavy, or for redundancy, use 2+ | `2` |
|  | `imageCredentials` | Information to access the Akamai docker registry |  |
|  | `imageCredentials.enabled` | If you do not have image pull secret already deployed in the Kubernetes cluster, change to `true` and fill registry credentials: registry, username, password, email. | `true` |
|  | `imageCredentials.registry` | URL of docker registry service | `us-central1-docker.pkg.dev` |
|  | `imageCredentials.namespaceRegistry` | Full path to the docker registry service and the images in the registry | `us-central1-docker.pkg.dev/noname-artifacts/nns-docker` |
|  | `imageCredentials.username` | Username to access docker registry |  |
|  | `imageCredentials.password` | Password to access docker registry |  |
|  | `imageCredentials.email` | Email address to access docker registry |  |
|  | `engine_platform.env.DATA_ORCHESTRATION_PUBLIC_HOSTNAME` | Use the API Security Server URL (with the FQDN). | Generated by API Security |
|  | `additionalInitContainers` | You can optionally add your oen init containers here. Enter `name`, `image`, and `command`. For example: `- name: myapp-container` `image: busynox:1.28` `command: ['sh', '-c', 'echo The app is running! &amp;&amp; sleep 3600']` |  |
|  | `proxy` | Proxy settings |  |
|  | `proxy.enabled` | Whether to enable a proxy (Boolean) |  |
|  | `proxy.env.NO_PROXY` | Enter the pods that should not use any configured proxy. For example, *"localhost,169.254.169.254,nginx,engine,nats,nats-jetstream,router"*. You may need to add this same line to **/etc/environment** and **/etc/systemd/system/k3s.service.env**, as follows: *NO_PROXY: "localhost,169.254.169.254,nginx,engine,nats,nats-jetstream,router"* Note: 169.254.169.254 is a special IP used by cloud providers, used to retrieve instance metadata specific to a instance. |  |
|  | `proxy.env.NONAME_HTTP_PROXY` | `&lt;host[:port]&gt;` | HTTP path to the proxy. The default port is 80. |
|  | `proxy.env.NONAME_HTTP_PROXY_BASIC_AUTH` | `&lt;username:password&gt;` | Adds the Proxy-Authorization header to the HTTP request. |
| istio |  |  |  |
|  | `enabled` | Whether to create Istio gateway. If `true`, then `ingress.enabled` must be `false`. | `false` |
|  | `host` | Istio hostname, for example `example.host.com` |  |
|  | `tls` | Istio credentialName and mode |  |
|  | `peerAuthentication` | Istio peer authentication mode: `PERMISSIVE`, `DISABLE`, `UNSET`. |  |
| ingress |  |  |  |
|  | `enabled` | Whether to create Ingress. Typically set to `true`. If `istio.enabled` is `true`, this must be `false`. | Generated by API Security |
|  | `ingressClassName` | Specifies which Ingress controller handles incoming requests for resources in your Kubernetes cluster. Use values like `nginx`, `gce`, `traefik`, `alb`, `kong` or the name of your custom controller. For more information, see [Ingress docs](https://kubernetes.io/blog/2020/04/02/improvements-to-the-ingress-api-in-kubernetes-1.18/#specifying-the-class-of-an-ingress) | `alb` |
|  | `annotations` | Configure Ingress annotations in this section. Use the [provided examples of annotations](/v368/docs/remote-engine-deployment-kubernetes#annotations-examples) or learn more in [Azure docs](https://learn.microsoft.com/en-us/azure/aks/load-balancer-standard), [GCP docs](https://cloud.google.com/kubernetes-engine/docs/concepts/service-load-balancer-parameters), or [AWS docs](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.7/). |  |
|  | `tls.hosts` | Hosts to allow TLS. See the [K8s docs](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls) |  |
|  | `tls.secretName` | Namespace secret name holding certificate. See the [K8s docs](https://kubernetes.io/docs/concepts/configuration/secret/#tls-secrets) |  |
|  | `hosts.host` | FQDN API Security Server URL, points to the Load Balancer | Generated by API Security |
|  | `hosts.host.paths.path` |  | `/` |
|  | `hosts.host.paths.pathType` | Each path in an Ingress must have a corresponding path type. | Generated by API Security |

### Annotations Examples

Here are examples of annotations for Amazon Load Balancer (ALB), Nginx, and Traefik that you can use as a reference to adjust them according to your specific ingress controller setup.

Amazon Load Balancer (ALB) example:

```
annotations:
  alb.ingress.kubernetes.io/scheme: "internet-facing"
  alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
  alb.ingress.kubernetes.io/ssl-redirect: '443'
  alb.ingress.kubernetes.io/target-type: "ip"
```

Nginx example:

```
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
  nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

Traefik example (Default Ingress Controller in API Security Server's K3s installation):

```
annotations:
  ingress.kubernetes.io/ssl-redirect: "true"
  traefik.ingress.kubernetes.io/router.entrypoints: websecure
```

## Appendix B: Troubleshooting

### First Step

Download the logs from the Engine page in the UI and examine them.

Then check the status of all pods: `kubectl get pods -n noname`

You should see something like:

```
NAME READY STATUS RESTARTS AGE
engine-6d99fc8477-hns6p 2/2 Running 0 66m
nginx-778995bb6f-h2zrj 1/1 Running 0 66m
...
```

All pods should be in a `Running` state. For a quick end-to-end validation, test `https://&lt;remote-engine-fqdn&gt;/engine` and expect a `200` response.

Redact Secrets Before Sharing

If you share command output with Akamai support, first redact any sensitive values - the output of `kubectl get secret -o yaml`, `helm get values --all`, rendered Helm templates, and any pod logs that contain registration keys, tokens, certificates, private keys, or private endpoints.

### Troubleshooting Order

Work from the deployment layer down to the application, so you identify the failing layer before chasing symptoms:

1. Validate the Helm release status.
2. Check pods, PVCs, services, ingress, and events.
3. If pods are `Pending`, check CPU, memory, quota, node capacity, and PVCs.
4. If images fail to pull, check registry access, image tags, and image pull secrets.
5. If pods crash or restart, check `describe`, current logs, previous logs, and the last termination reason.
6. If logs show registration, DNS, TLS, or timeout errors, validate outbound connectivity to the API Security Server.
7. If events show probe failures, test the health endpoint and tune probes through Helm values.
8. If pods are healthy but the Remote Engine URL does not work, troubleshoot ingress, ingress class, DNS, TLS, and service routing.

### Quick Symptom Reference

| Symptom | First command | Common cause |
| --- | --- | --- |
| Helm release is `pending-install`, `pending-upgrade`, or `failed` | `helm status &lt;release&gt; -n noname` | Pod not ready, PVC pending, image pull issue, probe failure |
| Pod is `Pending` | `kubectl describe pod &lt;pod&gt; -n noname` | Insufficient CPU/memory, quota, taints, affinity, unbound PVC |
| Pod is `ContainerCreating` | `kubectl describe pod &lt;pod&gt; -n noname` | Volume mount, CNI, or image pull delay |
| Pod is `ImagePullBackOff` / `ErrImagePull` | `kubectl describe pod &lt;pod&gt; -n noname` | Wrong image/tag, missing or invalid pull secret, registry access |
| Pod is `CrashLoopBackOff` | `kubectl logs &lt;pod&gt; -n noname --previous` | App crash, registration failure, bad config, OOM |
| Pod is `Running` but not `Ready` | `kubectl describe pod &lt;pod&gt; -n noname` | Readiness probe failure, registration incomplete |
| Remote Engine URL times out | `kubectl get ingress -n noname -o wide` | DNS, load balancer, firewall, ingress controller |
| Remote Engine URL returns 404 | `kubectl describe ingress -n noname` | Wrong ingress class, host/path mismatch, missing Host header |
| Remote Engine URL returns 502/503 | `kubectl get endpoints -n noname` | Service has no ready endpoints, backend pod not ready |

### Validate the Helm Release

Start with Helm before troubleshooting individual pods:

```
helm status <release> -n noname
helm history <release> -n noname
kubectl get all,pvc,ingress -n noname -o wide
kubectl get events -n noname --sort-by=.lastTimestamp | tail -150
```

Look for the lowest-level failing object (pod cannot schedule, PVC cannot bind, image cannot pull, workload exists but pod not ready, ingress created but has no address). After fixing the underlying issue, reconcile with `helm upgrade`.

Do Not Repeatedly Reinstall

Do not repeatedly uninstall and reinstall the release before you know what is blocking readiness. If you must remove the release, uninstall the Helm chart but do not delete the namespace; to delete the namespace after installation, contact Akamai support.

### Pod Is Pending: Quota, CPU, Memory, and Node Capacity

A Remote Engine can require large CPU and memory requests. A pod can stay `Pending` even when the cluster has enough total capacity if no single node can satisfy the request.

```
kubectl get resourcequota -n noname -o wide
kubectl get limitrange -n noname -o yaml
kubectl get nodes -o wide
kubectl top nodes
kubectl describe pod <pod> -n noname
```

Look for events such as `Insufficient cpu`, `Insufficient memory`, `exceeded quota`, `pod has unbound immediate PersistentVolumeClaims`, `node(s) had untolerated taint`, or `node(s) didn't match pod affinity/selector`.

Common fixes: increase the namespace CPU/memory quota; add or resize worker nodes; deploy to a node pool that can satisfy the engine pod requests; fix node selectors, tolerations, or affinity rules.

Note

Do not reduce the engine's resource requests or limits without guidance from Akamai support. Most clusters use auto-scaling; a large engine deployment can trigger a scale-up, and pods may remain `Pending` until the scaling event completes.

### PVC and Storage Issues

The Remote Engine requires persistent storage. If PVCs are `Pending`, the pods that depend on them will not start.

```
kubectl get pvc -n noname -o wide
kubectl describe pvc <pvc> -n noname
kubectl get storageclass
```

Look for a PVC stuck in `Pending`, no default `StorageClass`, a requested size that exceeds quota, an unsupported access mode, or a zone/topology mismatch. Fixes: set the correct `StorageClass` in your Helm values, increase the storage quota, use a storage class your cluster supports, and ensure dynamic provisioning works.

### Image Pull and Registry Issues

For private registries, the image pull secret and registry egress must be valid.

```
kubectl describe pod <pod> -n noname | sed -n '/Events:/,$p'
```

Look for a wrong image repository or tag, a missing or invalid image pull secret, a registry blocked by a firewall or proxy, or a mirror registry missing required images. Fixes: correct the image repository/tag in your Helm values, recreate the image pull secret, confirm node-level egress to the registry, and confirm all required images exist in your registry.

### CrashLoopBackOff or Restarting Pods

Collect the description, current logs, previous logs, and the last termination reason:

```
kubectl describe pod <pod> -n noname
kubectl logs <pod> -n noname --all-containers=true --tail=300 --timestamps
kubectl logs <pod> -n noname --all-containers=true --previous --tail=300 --timestamps
```

| Evidence | Meaning |
| --- | --- |
| `OOMKilled` | Container exceeded its memory limit, or the node was under memory pressure |
| Non-zero exit code | Application process exited with an error |
| Liveness probe failed | Kubelet restarted the container because the health check failed |
| Startup probe failed | Application did not become healthy within the startup window |
| Registration / DNS / TLS / timeout errors | Likely outbound connectivity to the API Security Server, or a certificate/proxy issue |

### Validate Outbound Connectivity to the Server

The Remote Engine must reach the API Security Server over outbound HTTPS to register and operate. If outbound access is blocked, the engine can fail registration and restart. Confirm the [required outbound endpoints](/docs/remote-engine-deployment-kubernetes#prerequisites) are reachable on TCP 443.

Test DNS and HTTPS from inside the namespace (use a temporary pod only if permitted in your environment):

```
kubectl run re-egress-test -n noname --rm -it --restart=Never --image=curlimages/curl -- sh
# inside the pod:
nslookup <server-hostname>
curl -vk --connect-timeout 5 --max-time 15 https://<server-hostname>:443/
```

| Result | Meaning |
| --- | --- |
| DNS lookup fails | DNS or cluster resolver issue |
| TCP timeout | Firewall, proxy, route, NAT, or network policy blocking egress |
| TLS handshake failure | TLS interception, proxy, certificate, SNI, or mTLS issue |
| HTTP 403 / 404 / redirect | Network path works; application auth/path may be expected |
| Connection refused | Wrong host/port, or upstream blocking |

If the namespace uses default-deny egress, allow at minimum: DNS to the cluster DNS service, and outbound TCP 443 to the [required Server and update endpoints](/docs/remote-engine-deployment-kubernetes#prerequisites).

### Health Check and Probe Failures

On slow clusters or slow storage, aggressive probe timeouts can restart a pod that would otherwise become healthy.

```
kubectl describe pod <pod> -n noname | sed -n '/Events:/,$p'
```

Look for `context deadline exceeded`, `connection refused`, or `HTTP/Startup/Liveness/Readiness probe failed`. Fixes: increase `timeoutSeconds`, `failureThreshold`, or `initialDelaySeconds` through Helm values and run `helm upgrade`; fix outbound Server connectivity if health depends on successful registration.

Note

Apply probe changes through Helm values, not `kubectl edit deployment` - manual edits are overwritten by the next Helm upgrade.

### Services and Endpoints

If pods are `Running` but the engine is not reachable internally or through ingress:

```
kubectl get svc,endpoints -n noname -o wide
kubectl describe svc <service> -n noname
```

If a service has no endpoints: backend pods are not ready, the service selector does not match pod labels, readiness probes are failing, or the service port/targetPort is wrong.

### Remote Engine URL, Ingress, and 404 Errors

Use this when all pods are deployed but the Remote Engine URL does not respond, times out, or returns 404/502/503.

**Check the ingress resource:**

```
kubectl get ingress -n noname -o wide
kubectl describe ingress -n noname
```

Validate that the ingress has an address, the host matches the Remote Engine FQDN, the backend service name and port are correct, the TLS secret is configured if HTTPS is required, and the ingress class is correct.

**Validate the ingress class** (a common cause is that the ingress object exists but no controller - or the wrong controller - is handling it):

```
kubectl get ingressclass
kubectl get ingress <ingress> -n noname -o jsonpath='{.spec.ingressClassName}{"\n"}'
```

`spec.ingressClassName` should match your cluster's intended controller. Do not assume the class is `nginx` - confirm the correct class with whoever owns the cluster. If it is wrong, fix it in your Helm values and redeploy.

**Test the URL with the correct Host header:**

```
curl -vk "https://<remote-engine-fqdn>/"
# testing directly against the load balancer IP/DNS:
curl -vk -H "Host: <remote-engine-fqdn>" "https://<load-balancer-ip-or-dns>/"
```

| Result | Likely area |
| --- | --- |
| DNS does not resolve | DNS record missing or pointing to the wrong load balancer |
| Connection timeout | Firewall, security group, route, load balancer, or ingress controller |
| TLS certificate mismatch | Wrong TLS secret/certificate, wrong SNI, or wrong hostname |
| 404 from ingress controller | Host/path mismatch or wrong ingress class/controller |
| 404 from backend app | Request reached the backend but the path is invalid |
| 502 / 503 | Ingress matched the service, but the service has no healthy endpoints |

### Validate the Deployment Foundation

Many issues trace back not to the visible error but to a mismatch in deployment assumptions. Before digging into logs or application errors, confirm:

- the Helm chart version matches the API Security Server version (see [Engine Support By Version](/v368/docs/engine-support-by-version));
- the correct `custom_values.yaml` is being used;
- the namespace, `StorageClass`, and ingress class match your cluster;
- NetworkPolicy, service mesh, proxy, and firewall rules allow the required connectivity;
- no manual changes were made after deployment that are not reflected in `custom_values.yaml`.

### Engine Fails to Connect Using Proxy

Most remote engines use a proxy to communicate with the management server. When the engine needs the communicate through a proxy, and the engine is offline, and the engine pod prints the following logs:

```
Connecting to dal at <noname_management_domain>
Stopping engine...
```

Try changing the environment variable **DATA_ORCHESTRATION_USE_NGINX** to **true**. This makes the engine route traffic through another pod that communicates using the proxy.

#### Proxy is Downgrading TLS Version

In Ubuntu 22, the default SSL library is libssl3. This library forbids unsafe renegotiation, by default.

Most remote engines use a proxy to communicate with the management server. Old proxies may try to downgrade the TLS version (from 1.3 to 1.2), causing unsafe renegotiation attempts. libssl will drop the connection and no traffic will get to the management: effectively, the engine will not work.

Here are some means to identify this problem:

- The engine cannot communicate with the server. In the UI, the engine appears as Offline/Partially Online.
- On the machine, run the following command in the engine container: `openssl s_client -connect &lt;management:port&gt; -proxy &lt;proxy:port&gt; -servername &lt;servername&gt;`. The output includes `Secure Renegotiation IS NOT supported`.
- If you have access to a PCAP file, you should see in Wireshark TLS alert in the the TLS handshake.

To resolve this:

1. Verify that the load balancer accepts TLS 1.3 traffic.
2. In the remote engine, set the environment variable **ENABLE_OPENSSL_CONF** to **true**.

If the above does not work, try the following:

1. Verify that the load balancer accepts TLS 1.3 traffic.
2. In the remote engine, create the text file **/data/configuration/openssl.cnf** as follows:

```
echo -e "openssl_conf = openssl_init\n\n[openssl_init]\nssl_conf = ssl_sect\n\n[ssl_sect]\nsystem_default = system_default_sect\n\n[system_default_sect]\nOptions = UnsafeLegacyRenegotiation" > /data/configuration/openssl.cnf
```
3. Set the environment variable **OPENSSL_CONF** to **/data/configuration/openssl.cnf**.

Processes API traffic within the user's environments and sends only the metadata of the API inventory, findings, and incidents, obfuscated samples, metrics, and models to the API Security Platform. Enables users to deploy multiple engines to transmit processes API traffic data.

Copies inbound and outbound traffic from your sources' ports/network interfaces and sends it to the API Security engine to perform out-of-band inspection. This solution is only supported where traffic is sent unencrypted, and thus requires the API Security engine to be deployed in the same network segment as the source.