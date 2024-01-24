# kube-prometheus-dashboards

`kube-prometheus-dashboards` is an open-source project aimed at providing improved dashboards to replace the built-in dashboards in [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack).

> ***Please note that these dashboards are based on the upstream dashboards from helm chart version 55.5.0 (December 2023). Changes to the upstream dashboards will NOT automatically be implemented.***

# Installing

To use `kube-prometheus-dashboards`, follow these installation steps:

1. Change your `kube-prometheus-stack` values.yaml to disable the installation of default dashboards:

```
    grafana:
        defaultDashboardsEnabled: false
```

2. Add the kube-prometheus-dashboards Helm repository:

```
helm repo add kube-prometheus-dashboards https://signicat.github.io/kube-prometheus-dashboards
helm repo update
```

3. Install the dashboards into the monitoring namespace, using the same configuration as for kube-prometheus-stack, but with dashboards enabled:

```
helm upgrade --install kube-prom-dash --namespace monitoring kube-prometheus-dashboards/kube-prometheus-dashboards-original-v2 --values my-kube-prometheus-stack-values.yaml --set grafana.defaultDashboardsEnabled=true
```
    
4. Optionally, restart Grafana if automatic dashboard reload doesn't work:

```
kubectl rollout restart -n monitoring deployment/grafana
```

### Uninstalling:

To uninstall `kube-prometheus-dashboards`, run the following command:

    helm uninstall --namespace monitoring kube-prom-dash

## Versions & features

### Version Original v2 (`kube-prometheus-dashboards-original-v2`)

- Control the creation of individual dashboards using `grafana.defaultDashboardsDisabled.x`.
- Shortened dashboard names to fit Grafana's search menu and dashboard title. ([Grafana GH Issue](https://github.com/grafana/grafana/issues/66635))
- Dashboard JSON data in ConfigMaps is formatted instead of being in one long line.
- Override default dashboard tags with grafana.overrideDefaultDashboardTag.
- Removes all interval panel settings.
- Removes all intervalFactor panel settings.

The primary goal is to eliminate the hard-coded `interval`(`1m`) and `intervalFactor`(`2`) settings that limit the resolution of rendered metrics. For example, a `rate()` requires at least two datapoints separated by 2 minutes, resulting in a maximum resolution of 4 minutes (2 * `interval` * `intervalFactor`). Although a [GH issue](https://github.com/prometheus-community/helm-charts/issues/2176) was created to address this, it didn't gain traction. The Grafana UI element for Resolution is deprecated and was [soft-hidden in Grafana 8.5](https://github.com/grafana/grafana/issues/48081) almost two years ago.

Additionally, some dashboards have been renamed for brevity, allowing more of the title to fit in the search menu and dashboard title. Common terms have been abbreviated:

- Kubernetes -> K8s
- Networking -> Net
- Compute Resources -> Compute

## Version: Original (`kube-prometheus-dashboards-original`)

- Control the creation of individual dashboards using `grafana.defaultDashboardsDisabled.x`.

# Detailed Usage and Caveats

Installation of these dashboards requires the `grafana.sidecar.dashboards` configuration to be enabled and properly set up.

You have the option to use the same `values.yaml` file for these charts as for `kube-prometheus-stack`, but remember to override `grafana.defaultDashboardsEnabled` to `false` for the installation of `kube-prometheus-stack` and `true` for `kube-prometheus-dashboards` chart installation. Alternatively, you can move the relevant values to a separate values file. Note that certain dashboards only render if specific conditions are met, such as `coreDns.enabled: true`.

## Grafana automatic dashboard reload not working

We have encountered an issue with automatic dashboard reloading not working, possibly due to the Grafana Admin API [only being available with Basic Auth](https://grafana.com/docs/grafana/latest/developers/http_api/admin/#admin-api), which we don't have enabled. There is a [related Grafana GH issue](https://github.com/grafana/helm-charts/issues/981) addressing this. As a workaround, we recommend restarting Grafana after making changes:

    kubectl rollout restart -n monitoring deployment/grafana

# Roadmap

The long term plan is to re-build some (or all) of the default dashboards:
- Using recent Grafana panel types
- Making use of a more diverse set of visualizations
- Improving queries and aggregations
- Improving navigation between dashboards

# Development

To render the chart for development, use the following command:

    helm template charts/kube-prometheus-dashboards-original-v2 --values my-values.yaml

## Version Original v2 (`kube-prometheus-dashboards-original-v2`)

This version is based on the Original version (see below). To remove `intervalFactor` and `interval`, use the following commands:

    sed -i.bak '/intervalFactor/d' charts/kube-prometheus-dashboards-original-v2/templates/*
    sed -i.bak '/"interval":/d' charts/kube-prometheus-dashboards-original-v2/templates/*

## Version: Original (`kube-prometheus-dashboards-original`)

In this version, we clone the original `kube-prometheus-stack` git repo from https://github.com/prometheus-community/helm-charts and copy select folders and files:

    cp -r ../../prometheus-community/helm-charts/charts/kube-prometheus-stack/templates/grafana/dashboards-1.14 charts/kube-prometheus-dashboards-original/templates
    cp ../../prometheus-community/helm-charts/charts/kube-prometheus-stack/templates/_helpers.tpl charts/kube-prometheus-dashboards-original/templates
    cp ../../prometheus-community/helm-charts/charts/kube-prometheus-stack/values.yaml charts/kube-prometheus-dashboards-original/

Changes made include:

    Adding `grafana.defaultDashboardsDisabled.x` to values.
    Adding a clause `(not .Values.grafana.defaultDashboardsDisabled.x)` to every template file.
    To allow installation side-by-side with the "legacy" dashboards, we rename the filename in each ConfigMap by adding -v1 to avoid conflicts. Similarly, we add -v1 as a suffix to existing dashboard uids. This ensures that new dashboards won't be loaded if the UID collides with an existing one, making it easier to track lineage and possibly trace down the original sources. Note that some dashboards may be missing a uid, or it may be blank, which is not a problem as Grafana will generate one for them on load.
