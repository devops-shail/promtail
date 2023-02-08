promtail
Version: 6.8.2 Type: application AppVersion: 2.7.2

Promtail is an agent which ships the contents of local logs to a Loki instance

Source Code
https://github.com/grafana/loki
https://grafana.com/oss/loki/
https://grafana.com/docs/loki/latest/
Chart Repo
Add the following repo to use the chart:

helm repo add grafana https://grafana.github.io/helm-charts
Upgrading
A major chart version change indicates that there is an incompatible breaking change needing manual actions.

From Chart Versions >= 3.0.0
Customizeable initContainer added.
From Chart Versions < 3.0.0
Notable Changes
Helm 3 is required
Labels have been updated to follow the official Kubernetes label recommendations
The default scrape configs have been updated to take new and old labels into consideration
The config file must be specified as string which can be templated. See below for details
The config file is now stored in a Secret and no longer in a ConfigMap because it may contain sensitive data, such as basic auth credentials
Due to the label changes, an existing installation cannot be upgraded without manual interaction. There are basically two options:

Option 1
Uninstall the old release and re-install the new one. There will be no data loss. Promtail will cleanly shut down and write the positions.yaml. The new release which will pick up again from the existing positions.yaml.

Option 2
Add new selector labels to the existing pods:

kubectl label pods -n <namespace> -l app=promtail,release=<release> app.kubernetes.io/name=promtail app.kubernetes.io/instance=<release>
Perform a non-cascading deletion of the DaemonSet which will keep the pods running:

kubectl delete daemonset -n <namespace> -l app=promtail,release=<release> --cascade=false
Perform a regular Helm upgrade on the existing release. The new DaemonSet will pick up the existing pods and perform a rolling upgrade.

  Configuration
The config file for Promtail must be configured as string. This is necessary because the contents are passed through the tpl function. With this, the file can be templated and assembled from reusable YAML snippets. It is common to have multiple kubernetes_sd_configs that, in turn, usually need the same pipeline_stages. Thus, extracting reusable snippets helps reduce redundancy and avoid copy/paste errors. See `values.yaml´ for details. Also, the following examples make use of this feature.

  Syslog Support
extraPorts:
  syslog:
    name: tcp-syslog
    containerPort: 1514
    service:
      port: 80
      type: LoadBalancer
      externalTrafficPolicy: Local
      loadBalancerIP: 123.234.123.234

config:
  snippets:
    extraScrapeConfigs: |
      # Add an additional scrape config for syslog
      - job_name: syslog
        syslog:
          listen_address: 0.0.0.0:{{ .Values.extraPorts.syslog.containerPort }}
          labels:
            job: syslog
        relabel_configs:
          - source_labels:
              - __syslog_message_hostname
            target_label: hostname

          # example label values: kernel, CRON, kubelet
          - source_labels:
              - __syslog_message_app_name
            target_label: app

          # example label values: debug, notice, informational, warning, error
          - source_labels:
              - __syslog_message_severity
            target_label: level

For additional reference, please refer to Promtail's docs:

https://grafana.com/docs/loki/latest/clients/promtail/configuration/




For additional reference, please refer to Promtail's docs:

https://grafana.com/docs/loki/latest/clients/promtail/configuration/

Syslog Support
extraPorts:
  syslog:
    name: tcp-syslog
    containerPort: 1514
    service:
      port: 80
      type: LoadBalancer
      externalTrafficPolicy: Local
      loadBalancerIP: 123.234.123.234

config:
  snippets:
    extraScrapeConfigs: |
      # Add an additional scrape config for syslog
      - job_name: syslog
        syslog:
          listen_address: 0.0.0.0:{{ .Values.extraPorts.syslog.containerPort }}
          labels:
            job: syslog
        relabel_configs:
          - source_labels:
              - __syslog_message_hostname
            target_label: hostname

          # example label values: kernel, CRON, kubelet
          - source_labels:
              - __syslog_message_app_name
            target_label: app

          # example label values: debug, notice, informational, warning, error
          - source_labels:
              - __syslog_message_severity
            target_label: level
Find additional source labels in the Promtail's docs:

https://grafana.com/docs/loki/latest/clients/promtail/configuration/#syslog

Journald Support
config:
  snippets:
    extraScrapeConfigs: |
      # Add an additional scrape config for syslog
      - job_name: journal
        journal:
          path: /var/log/journal
          max_age: 12h
          labels:
            job: systemd-journal
        relabel_configs:
          - source_labels:
              - __journal__hostname
            target_label: hostname

          # example label values: kubelet.service, containerd.service
          - source_labels:
              - __journal__systemd_unit
            target_label: unit

          # example label values: debug, notice, info, warning, error
          - source_labels:
              - __journal_priority_keyword
            target_label: level

# Mount journal directory and machine-id file into promtail pods
extraVolumes:
  - name: journal
    hostPath:
      path: /var/log/journal
  - name: machine-id
    hostPath:
      path: /etc/machine-id

extraVolumeMounts:
  - name: journal
    mountPath: /var/log/journal
    readOnly: true
  - name: machine-id
    mountPath: /etc/machine-id
    readOnly: true
Find additional configuration options in Promtail's docs:

https://grafana.com/docs/loki/latest/clients/promtail/configuration/#journal

More journal source labels can be found here https://www.freedesktop.org/software/systemd/man/systemd.journal-fields.html.

Note that each message from the journal may have a different set of fields and software may write an arbitrary set of custom fields for their logged messages. (related issue)

The machine-id needs to be available in the container as it is required for scraping. This is described in Promtail's scraping docs:

https://grafana.com/docs/loki/latest/clients/promtail/scraping/#journal-scraping-linux-only

Push API Support
extraPorts:
  httpPush:
    name: http-push
    containerPort: 3500
  grpcPush:
    name: grpc-push
    containerPort: 3600

config:
  file: |
    server:
      log_level: {{ .Values.config.logLevel }}
      http_listen_port: {{ .Values.config.serverPort }}

    clients:
      - url: {{ .Values.config.lokiAddress }}

    positions:
      filename: /run/promtail/positions.yaml

    scrape_configs:
      {{- tpl .Values.config.snippets.scrapeConfigs . | nindent 2 }}

      - job_name: push1
        loki_push_api:
          server:
            http_listen_port: {{ .Values.extraPorts.httpPush.containerPort }}
            grpc_listen_port: {{ .Values.extraPorts.grpcPush.containerPort }}
          labels:
            pushserver: push1
Customize client config options
By default, promtail send logs scraped to loki server at http://loki-gateway/loki/api/v1/push. If you want to customize clients or add additional options to loki, please use the clients section. For example, to enable HTTP basic auth and include OrgID header, you can use:

config:
  clients:
    - url: http://loki.server/loki/api/v1/push
      tenant_id: 1
      basic_auth:
        username: loki
        password: secret
