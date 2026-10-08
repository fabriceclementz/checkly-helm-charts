# Checkly Agent Helm Chart

This helm chart deploys the Checkly agent to create your own [private location](https://www.checklyhq.com/docs/private-locations/).

## Prerequisites

Create a private location as described [here](https://www.checklyhq.com/docs/private-locations/#configuring-a-private-location) to get a agent api key.

## Usage

[Helm](https://helm.sh) must be installed to use the charts. Please refer to Helm's [documentation](https://helm.sh/docs) to get started.

Once Helm has been set up correctly, add the repo as follows:

    helm repo add checkly https://checkly.github.io/helm-charts

If you had already added this repo earlier, run `helm repo update` to retrieve the latest versions of the packages.

To install the agent chart:

    helm upgrade --install my-agent checkly/agent --set apiKeySecret.apiKey=pl_...

To uninstall the chart:

    helm uninstall my-agent

See [https://github.com/checkly/helm-charts](https://github.com/checkly/helm-charts) for more info and available charts.

## Using it with Terraform

```
resource "helm_release" "checkly_agent" {
  name       = "checkly-agent"
  repository = "https://checkly.github.io/helm-charts"
  chart      = "agent"
  version    = "0.3.0"

  # https://github.com/checkly/helm-charts/blob/main/charts/agent/values.yaml
  values = [
    <<-EOT
    # add your values.yaml conf here
    EOT
  ]
}
```

## Autoscaling with KEDA

The chart can create a [KEDA](https://keda.sh) `ScaledObject` that scales the agents based on the number of queued and in-flight check runs, as described in the [autoscaling documentation](https://www.checklyhq.com/docs/platform/private-locations/autoscaling/).

Prerequisites:

- KEDA installed in the cluster
- Prometheus V2 metrics ingestion enabled, with the `checkly_private_location_check_runs` metric available in a Prometheus-compatible server

```
env:
  JOB_CONCURRENCY: 5

autoscaling:
  enabled: true
  minReplicaCount: 2
  maxReplicaCount: 10
  privateLocationSlugName: my-private-location
  prometheus:
    serverAddress: http://prometheus-k8s.monitoring.svc.cluster.local:9090
```

When autoscaling is enabled:

- `spec.replicas` is not set on the Deployment, so the replica count is owned by the HPA managed by KEDA and `replicaCount` is ignored. This avoids conflicts between Helm and the HPA, for instance with server-side apply.
- The scaling threshold defaults to `env.JOB_CONCURRENCY` (or `1`), as recommended. It can be overridden with `autoscaling.threshold`.
- `terminationGracePeriodSeconds` defaults to `330` so in-flight checks can complete on scaled-down pods.

Use `autoscaling.query` to override the default query, and `autoscaling.prometheus.authenticationRef` to reference a KEDA `TriggerAuthentication` if your Prometheus server requires authentication. See [values.yaml](values.yaml) for all options.

## Alternative ways to set the agent API Key

Instead of setting `apiKeySecret.apiKey` you can also choose an existing secret with the following options

```
apiKeySecret:
  create: false
  name: <NAME_OF_EXISTING_SECRET>
```

or create the secret with `extraManifests`

```
apiKeySecret:
  create: false
  name: checkly-agent-secret

extraManifests:
  - apiVersion: external-secrets.io/v1beta1
    kind: ExternalSecret
    metadata:
      name: checkly-agent-secret
      namespace: monitoring
    spec:
      target:
        name: my-checkly-secret-in-aws
```
