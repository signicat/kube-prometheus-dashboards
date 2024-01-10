# kube-prometheus-dashboards

Improved dashboards replacing the built-in dashboards in [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack).

> PS: The dashboards are based on upstream dashboards from helm chart version 55.5.0 (December 2023). Changes to dashboards upstream will NOT automatically be implemented.

# Installing

Change your `kube-prometheus-stack` values.yaml to disable installation of default dashboards:

    grafana:
        defaultDashboardsEnabled: false

    # Add helm repository
    helm repo add .....
    # Install dashboards into `monitoring` namespace, using the same config as for `kube-prometheus-stack`, but with dashboards enabled:
    helm upgrade --install kube-prom-dash --namespace monitoring charts/kube-prometheus-dashboards-original-v2 --values my-kube-prometheus-stack-values.yaml --set grafana.defaultDashboardsEnabled=true
    
    # Optionally restart grafana if automatic dashboard reload doesn't work
    kubectl rollout restart -n monitoring deployment/grafana

Uninstalling:

    helm uninstall --namespace monitoring kube-prom-dash

## Versions & features

### Version Original v2 (`kube-prometheus-dashboards-original-v2`)

- Creation of individual dashboards can be controlled by `grafana.defaultDashboardsDisabled.x`.
- Dashboard names shortened to fit Grafana search menu and dashboard title. ([Grafana GH Issue](https://github.com/grafana/grafana/issues/66635))
- Dashboard JSON data in ConfigMaps formatted instead of one long line.
- Override default dashboard tags with `grafana.overrideDefaultDashboardTag`.
- Removes all `interval` panel settings.
- Removes all `intervalFactor` panel settings.

The main purpose is getting rid of the hard coded `interval`(`1m`) and `intervalFactor`(`2`) settings that are present on most panels. They will artificially limit the resolution of rendered metrics. For example a `rate()` requires at least two datapoints now 2 minutes apart, meaning any rates have a maximum resolution of 4 minutes (2 * `interval` * `intervalFactor`). I created a [GH issue about this](https://github.com/prometheus-community/helm-charts/issues/2176) a long time ago but it never gained traction. The actual Grafana UI element for Resolution is deprecated and was [soft-hidden in Grafana 8.5](https://github.com/grafana/grafana/issues/48081), almost two years ago.

Some dashboards have also been renamed to shorten them so more of the title fits in the search menu and dashboard title. There is an [open Grafana GH issue](https://github.com/grafana/grafana/issues/66635) about this as well. The general rule for renaming is abbreviating a few common terms:

- Kubernetes -> K8s
- Networking -> Net
- Compute Resources -> Compute


## Version: Original (`kube-prometheus-dashboards-original`)

- Creation of individual dashboards can be controlled by `grafana.defaultDashboardsDisabled.x`.

# Detailed usage and caveats

Installation of these dashboards require the `grafana.sidecar.dashboards` configuration to be enabled and set up properly.

You can choose whether you want to use the same `values.yaml` file for these charts as for `kube-prometheus-stack`, but remember to override so that `grafana.defaultDashboardsEnabled` is `false` on installation of `kube-prometheus-stack` and `true` for `kube-prometheus-dashboards` chart installation. Or you can move the relevant values to a separate values file. Be aware that there is logic in certain dashboards so they only render if for example `coreDns.enabled: true` is present.

We also experience an issue with automatic dashboard reloading not working. This is probably due to the Grafana Admin API [only being available with Basic Auth](https://grafana.com/docs/grafana/latest/developers/http_api/admin/#admin-api), which we don't have enabled. There is a [related Grafana GH issue](https://github.com/grafana/helm-charts/issues/981) about this. We work around this by restarting Grafana after making changes:

    kubectl rollout restart -n monitoring deployment/grafana


# Development

Render chart for development:

    helm template charts/kube-prometheus-dashboards-original-v2 --values my-values.yaml

## Version Original v2 (`kube-prometheus-dashboards-original-v2`)

The base for this version is the Original version, see below.

We remove `intervalFactor` and `interval` by:

    sed -i.bak '/intervalFactor/d' charts/kube-prometheus-dashboards-original-v2/templates/*
    sed -i.bak '/"interval":/d' charts/kube-prometheus-dashboards-original-v2/templates/*

## Version: Original (`kube-prometheus-dashboards-original`)

We clone the original kube-prometheus-stack git repo from https://github.com/prometheus-community/helm-charts and copy a few select folders and files:

    cp -r ../../prometheus-community/helm-charts/charts/kube-prometheus-stack/templates/grafana/dashboards-1.14 charts/kube-prometheus-dashboards-original/templates
    cp ../../prometheus-community/helm-charts/charts/kube-prometheus-stack/templates/_helpers.tpl charts/kube-prometheus-dashboards-original/templates
    cp ../../prometheus-community/helm-charts/charts/kube-prometheus-stack/values.yaml charts/kube-prometheus-dashboards-original/

Changes:
- Add grafana.defaultDashboardsDisabled.x to values.
- Add (not .Values.grafana.defaultDashboardsDisabled.x) clause to every template file.
- In order for this to be able to be installed side-by-side with the "legacy" dashboards two more changes are needed:
  - Rename the filename in each ConfigMap. The dashboards are stored in the sidecars in one folder based on the filename. So if the name is the same as the old who knows what's going to happen. For now in this v1 chart the renaming is by adding `-v1` so `alertmanager-overview.json` to `alertmanager-overview-v1.json`.
  - Also added `-v1` as a suffix to existing dashboard `uid`s. New dashboards won't be loaded if the UID collides with an existing UID. Keeping the old UID with a suffix makes it easier to track lineage and maybe track down the original sources. PS: Some dashboards are either missing `uid` or it is blank. That's not a problem since Grafana will generate one for them on load.

