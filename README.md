# kube-prometheus-dashboards
Improved dashboards for replacing the built-in dashboards in kube-prometheus-stack

## Why

## Version Original

PS: Not synced with upstream. Forked from chart version 55.5.0.

Drop in replacement for kube-prometheus-stack dashboards. Use the same values.yaml file as for kube-prometheus-stack.

### How it's made

    cp -r ../../prometheus-community/helm-charts/charts/kube-prometheus-stack/templates/grafana/dashboards-1.14 charts/kube-prometheus-dashboards-original/templates

    cp ../../prometheus-community/helm-charts/charts/kube-prometheus-stack/templates/_helpers.tpl charts/kube-prometheus-dashboards-original/templates

    cp ../../prometheus-community/helm-charts/charts/kube-prometheus-stack/values.yaml charts/kube-prometheus-dashboards-original/

    helm template charts/kube-prometheus-dashboards-original

Added grafana.defaultDashboardsDisabled to values
Added (not .Values.grafana.defaultDashboardsDisabled.proxy) clause to every template file


## Version Original v2

Copy of kube-prometheus-dashboards-original with a few improvements:
- template files formatted


helm template charts/kube-prometheus-dashboards-original-v2 --values charts/kube-prometheus-dashboards-original-v2/values-test.yaml

helm upgrade --install new-prom-dash --namespace metrics charts/kube-prometheus-dashboards-original-v2 --values charts/kube-prometheus-dashboards-original-v2/values-test.yaml

helm uninstall --namespace metrics new-prom-dash

## Version 3 and above



---

    {{`{

"editable":`}}{{ .Values.grafana.defaultDashboardsEditable}}{{`,

            }`}}



"hide":`}}{{ if .Values.grafana.sidecar.dashboards.multicluster.global.enabled }}0{{ else }}2{{ end }}{{`,



---
etcd dashboard missing editable config
same for coredns


---


Tracking dashboard usage on open source grafana?

Custom grafana logo for tracking pixel
Custom CSS?
Google analytics?

