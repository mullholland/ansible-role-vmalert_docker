# [Ansible role vmalert_docker](#vmalert_docker)

Installs and configures vmalert container based on official vmalert docker container

|GitHub|Downloads|Version|
|------|---------|-------|
|[![github](https://github.com/mullholland/ansible-role-vmalert_docker/actions/workflows/molecule.yml/badge.svg)](https://github.com/mullholland/ansible-role-vmalert_docker/actions/workflows/molecule.yml)|[![downloads](https://img.shields.io/ansible/role/d/mullholland/vmalert_docker)](https://galaxy.ansible.com/mullholland/vmalert_docker)|[![Version](https://img.shields.io/github/release/mullholland/ansible-role-vmalert_docker.svg)](https://github.com/mullholland/ansible-role-vmalert_docker/releases/)|
## [Example Playbook](#example-playbook)

This example is taken from [`molecule/default/converge.yml`](https://github.com/mullholland/ansible-role-vmalert_docker/blob/master/molecule/default/converge.yml) and is tested on each push, pull request and release.

```yaml
---
- name: Converge
  hosts: all
  become: true
  gather_facts: true
  vars:
    adguardhome_docker_config:
      tls:
        options:
          modern:
            minVersion: "VersionTLS13"
            sniStrict: true

  roles:
    - role: "mullholland.vmalert_docker"
```

The machine needs to be prepared. In CI this is done using [`molecule/default/prepare.yml`](https://github.com/mullholland/ansible-role-vmalert_docker/blob/master/molecule/default/prepare.yml):

```yaml
---
- name: Prepare
  hosts: all
  become: true
  gather_facts: true
  vars:
    pip_packages:
      - "docker"

  roles:
    - role: mullholland.docker
    - role: mullholland.repository_epel
    - role: mullholland.pip
```



## [Role Variables](#role-variables)

The default values for the variables are set in [`defaults/main.yml`](https://github.com/mullholland/ansible-role-vmalert_docker/blob/master/defaults/main.yml):

```yaml
---
# General config
vmalert_docker_network_name: "web"
vmalert_docker_base_path: "/opt"
vmalert_docker_timezone: "Europe/Berlin"

# User/Group of the stack. Everything is mapped to this, instead of root.
vmalert_docker_user: "homelab"
vmalert_docker_uid: "900"
vmalert_docker_group: "homelab"
vmalert_docker_gid: "900"
vmalert_docker_user_system: true

# which container version to install
# https://hub.docker.com/r/victoriametrics/vmalert/tags
# can also be latest
vmalert_docker_version: "victoriametrics/vmalert:latest"

# https://docs.victoriametrics.com/vmalert/#advanced-usage
vmalert_docker_commands:
  - "--datasource.url=http://victoriametrics:8428/"
  - "--remoteRead.url=http://victoriametrics:8428/"
  - "--remoteWrite.url=http://victoriametrics:8428/"
  - "--notifier.url=http://alertmanager:9093/"
  - "--rule=/etc/alerts/*.yml"
  # display source of alerts in grafana
  - "--external.url=http://grafana:3000"  # grafana container

vmalert_docker_environment_variables: []
vmalert_docker_volumes:
  - "{{ vmalert_docker_base_path }}/vmalert/alerts:/etc/alerts"

# which port to expose. can be empty
vmalert_docker_ports:
  - "8880:8880"

vmalert_docker_labels:
  - "traefik.enable=false"
# - "traefik.enable=true"
# - "traefik.http.routers.vmalert.entryPoints=https"
# - "traefik.http.routers.vmalert.rule=Host(`vmalert.example.com`)"
# - "traefik.http.routers.vmalert.tls.certResolver=letsEncrypt"
# - "traefik.http.routers.vmalert.tls=true"
#  - "traefik.http.routers.vmalert.service=vmalert"
#  - "traefik.http.routers.vmalert.loadbalancer.server.port=80"


# Example alerts are inspired by https://github.com/VictoriaMetrics/VictoriaMetrics/blob/master/deployment/docker/docker-compose.yml
# Contains default list of alerts for VictoriaMetrics single server.
# The alerts below are just recommendations and may require some updates
# and threshold calibration according to every specific setup.
vmalert_docker_alerts_vmsingle:
  - name: vmsingle
    interval: 30s
    concurrency: 2
    rules:
      - alert: DiskRunsOutOfSpaceIn3Days
        expr: '{% raw %}vm_free_disk_space_bytes / ignoring(path)( rate(vm_rows_added_to_storage_total[1d]) * scalar(sum(vm_data_size_bytes{type!~"indexdb.*"}) / sum(vm_rows{type!~"indexdb.*"}))) < 3 * 24 * 3600 > 0{% endraw %}'
        for: 30m
        labels:
          severity: critical
        annotations:
          # dashboard: "http://localhost:3000/d/wNf0q_kZk?viewPanel=73&var-instance={{ $labels.instance }}"
          summary: '{% raw %}Instance {{ $labels.instance }} will run out of disk space soon{% endraw %}'
          description: '{% raw %}Taking into account current ingestion rate, free disk space will be enough only for {{ $value | humanizeDuration }} on instance {{ $labels.instance }}.\n Consider to limit the ingestion rate, decrease retention or scale the disk space if possible.{% endraw %}'
      - alert: DiskRunsOutOfSpace
        expr: '{% raw %}sum(vm_data_size_bytes) by(job, instance) / (sum(vm_free_disk_space_bytes) by(job, instance) + sum(vm_data_size_bytes) by(job, instance)) > 0.8{% endraw %}'
        for: 30m
        labels:
          severity: critical
        annotations:
          # dashboard: "http://localhost:3000/d/wNf0q_kZk?viewPanel=53&var-instance={{ $labels.instance }}"
          summary: '{% raw %}Instance {{ $labels.instance }} (job={{ $labels.job }}) will run out of disk space soon{% endraw %}'
          description: '{% raw %}Disk utilisation on instance {{ $labels.instance }} is more than 80%.\n Having less than 20% of free disk space could cripple merges processes and overall performance. Consider to limit the ingestion rate, decrease retention or scale the disk space if possible.{% endraw %}'
      - alert: RequestErrorsToAPI
        expr: '{% raw %}increase(vm_http_request_errors_total[5m]) > 0{% endraw %}'
        for: 15m
        labels:
          severity: warning
        annotations:
          # dashboard: "http://localhost:3000/d/wNf0q_kZk?viewPanel=35&var-instance={{ $labels.instance }}"
          summary: '{% raw %}Too many errors served for path {{ $labels.path }} (instance {{ $labels.instance }}){% endraw %}'
          description: '{% raw %}Requests to path {{ $labels.path }} are receiving errors. Please verify if clients are sending correct requests.{% endraw %}'
      - alert: RowsRejectedOnIngestion
        expr: '{% raw %}rate(vm_rows_ignored_total[5m]) > 0{% endraw %}'
        for: 15m
        labels:
          severity: warning
        annotations:
          # dashboard: "http://localhost:3000/d/wNf0q_kZk?viewPanel=58&var-instance={{ $labels.instance }}"
          summary: '{% raw %}Some rows are rejected on \"{{ $labels.instance }}\" on ingestion attempt{% endraw %}'
          description: '{% raw %}VM is rejecting to ingest rows on \"{{ $labels.instance }}\" due to the following reason: \"{{ $labels.reason }}\"{% endraw %}'
      - alert: TooHighChurnRate
        expr: '{% raw %}(sum(rate(vm_new_timeseries_created_total[5m])) by(instance) / sum(rate(vm_rows_inserted_total[5m])) by (instance)) > 0.1{% endraw %}'
        for: 15m
        labels:
          severity: warning
        annotations:
          # dashboard: "http://localhost:3000/d/wNf0q_kZk?viewPanel=66&var-instance={{ $labels.instance }}"
          summary: '{% raw %}Churn rate is more than 10% on \"{{ $labels.instance }}\" for the last 15m{% endraw %}'
          description: '{% raw %}VM constantly creates new time series on \"{{ $labels.instance }}\".\n This effect is known as Churn Rate.\n High Churn Rate tightly connected with database performance and may result in unexpected OOMs or slow queries."{% endraw %}'
      - alert: TooHighChurnRate24h
        expr: '{% raw %}sum(increase(vm_new_timeseries_created_total[24h])) by(instance) > (sum(vm_cache_entries{type="storage/hour_metric_ids"}) by(instance) * 3){% endraw %}'
        for: 15m
        labels:
          severity: warning
        annotations:
          # dashboard: "http://localhost:3000/d/wNf0q_kZk?viewPanel=66&var-instance={{ $labels.instance }}"
          summary: '{% raw %}Too high number of new series on \"{{ $labels.instance }}\" created over last 24h{% endraw %}'
          description: '{% raw %}The number of created new time series over last 24h is 3x times higher than current number of active series on \"{{ $labels.instance }}\".\n This effect is known as Churn Rate.\n High Churn Rate tightly connected with database performance and may result in unexpected OOMs or slow queries.{% endraw %}'
      - alert: TooHighSlowInsertsRate
        expr: '{% raw %}(sum(rate(vm_slow_row_inserts_total[5m])) by(instance) / sum(rate(vm_rows_inserted_total[5m])) by (instance)) > 0.05{% endraw %}'
        for: 15m
        labels:
          severity: warning
        annotations:
          # dashboard: "http://localhost:3000/d/wNf0q_kZk?viewPanel=68&var-instance={{ $labels.instance }}"
          summary: '{% raw %}Percentage of slow inserts is more than 5% on \"{{ $labels.instance }}\" for the last 15m{% endraw %}'
          description: '{% raw %}High rate of slow inserts on \"{{ $labels.instance }}\" may be a sign of resource exhaustion for the current load. It is likely more RAM is needed for optimal handling of the current number of active time series. See also https://github.com/VictoriaMetrics/VictoriaMetrics/issues/3976#issuecomment-1476883183{% endraw %}'
      - alert: LabelsLimitExceededOnIngestion
        expr: '{% raw %}increase(vm_metrics_with_dropped_labels_total[5m]) > 0{% endraw %}'
        for: 15m
        labels:
          severity: warning
        annotations:
          # dashboard: "http://localhost:3000/d/wNf0q_kZk?viewPanel=74&var-instance={{ $labels.instance }}"
          summary: '{% raw %}Metrics ingested in ({{ $labels.instance }}) are exceeding labels limit{% endraw %}'
          description: '{% raw %}VictoriaMetrics limits the number of labels per each metric with `-maxLabelsPerTimeseries` command-line flag.\n This prevents from ingesting metrics with too many labels. Please verify that `-maxLabelsPerTimeseries` is configured correctly or that clients which send these metrics arent misbehaving.{% endraw %}'

# Contains default list of alerts for vmagent service.
# The alerts below are just recommendations and may require some updates
# and threshold calibration according to every specific setup.
vmalert_docker_alerts_vmagent:
  - name: vmagent
    interval: 30s
    concurrency: 2
    rules:
      - alert: PersistentQueueIsDroppingData
        expr: '{% raw %}sum(increase(vm_persistentqueue_bytes_dropped_total[5m])) without (path) > 0{% endraw %}'
        for: 10m
        labels:
          severity: critical
        annotations:
          # dashboard: "http://localhost:3000/d/G7Z9GzMGz?viewPanel=49&var-instance={{ $labels.instance }}"
          summary: '{% raw %}Instance {{ $labels.instance }} is dropping data from persistent queue{% endraw %}'
          description: '{% raw %}Vmagent dropped {{ $value | humanize1024 }} from persistent queue on instance {{ $labels.instance }} for the last 10m.{% endraw %}'
      - alert: RejectedRemoteWriteDataBlocksAreDropped
        expr: '{% raw %}sum(increase(vmagent_remotewrite_packets_dropped_total[5m])) without (url) > 0{% endraw %}'
        for: 15m
        labels:
          severity: warning
        annotations:
          # dashboard: "http://localhost:3000/d/G7Z9GzMGz?viewPanel=79&var-instance={{ $labels.instance }}"
          summary: '{% raw %}Job \"{{ $labels.job }}\" on instance {{ $labels.instance }} drops the rejected by remote-write server data blocks. Check the logs to find the reason for rejects.{% endraw %}'
      - alert: TooManyScrapeErrors
        expr: '{% raw %}increase(vm_promscrape_scrapes_failed_total[5m]) > 0{% endraw %}'
        for: 15m
        labels:
          severity: warning
        annotations:
          # dashboard: "http://localhost:3000/d/G7Z9GzMGz?viewPanel=31&var-instance={{ $labels.instance }}"
          summary: '{% raw %}Job \"{{ $labels.job }}\" on instance {{ $labels.instance }} fails to scrape targets for last 15m{% endraw %}'
      - alert: TooManyWriteErrors
        expr: '{% raw %}(sum(increase(vm_ingestserver_request_errors_total[5m])) without (name,net,type) + sum(increase(vmagent_http_request_errors_total[5m])) without (path,protocol)) > 0{% endraw %}'
        for: 15m
        labels:
          severity: warning
        annotations:
          # dashboard: "http://localhost:3000/d/G7Z9GzMGz?viewPanel=77&var-instance={{ $labels.instance }}"
          summary: '{% raw %}Job \"{{ $labels.job }}\" on instance {{ $labels.instance }} responds with errors to write requests for last 15m.{% endraw %}'
      - alert: TooManyRemoteWriteErrors
        expr: '{% raw %}rate(vmagent_remotewrite_retries_count_total[5m]) > 0{% endraw %}'
        for: 15m
        labels:
          severity: warning
        annotations:
          # dashboard: "http://localhost:3000/d/G7Z9GzMGz?viewPanel=61&var-instance={{ $labels.instance }}"
          summary: '{% raw %}Job \"{{ $labels.job }}\" on instance {{ $labels.instance }} fails to push to remote storage{% endraw %}'
          description: '{% raw %}Vmagent fails to push data via remote write protocol to destination \"{{ $labels.url }}\"\n Ensure that destination is up and reachable.{% endraw %}'
      - alert: RemoteWriteConnectionIsSaturated
        expr: '{% raw %}(rate(vmagent_remotewrite_send_duration_seconds_total[5m]) / vmagent_remotewrite_queues) > 0.9{% endraw %}'
        for: 15m
        labels:
          severity: warning
        annotations:
          # dashboard: "http://localhost:3000/d/G7Z9GzMGz?viewPanel=84&var-instance={{ $labels.instance }}"
          summary: '{% raw %}Remote write connection from \"{{ $labels.job }}\" (instance {{ $labels.instance }}) to {{ $labels.url }} is saturated{% endraw %}'
          description: '{% raw %}The remote write connection between vmagent \"{{ $labels.job }}\" (instance {{ $labels.instance }}) and destination \"{{ $labels.url }}\" is saturated by more than 90% and vmagent wont be able to keep up.\n This usually means that `-remoteWrite.queues` command-line flag must be increased in order to increase the number of connections per each remote storage.{% endraw %}'
      - alert: PersistentQueueForWritesIsSaturated
        expr: '{% raw %}rate(vm_persistentqueue_write_duration_seconds_total[5m]) > 0.9{% endraw %}'
        for: 15m
        labels:
          severity: warning
        annotations:
          # dashboard: "http://localhost:3000/d/G7Z9GzMGz?viewPanel=98&var-instance={{ $labels.instance }}"
          summary: '{% raw %}Persistent queue writes for instance {{ $labels.instance }} are saturated{% endraw %}'
          description: '{% raw %}Persistent queue writes for vmagent \"{{ $labels.job }}\" (instance {{ $labels.instance }}) are saturated by more than 90% and vmagent wont be able to keep up with flushing data on disk. In this case, consider to decrease load on the vmagent or improve the disk throughput.{% endraw %}'
      - alert: PersistentQueueForReadsIsSaturated
        expr: '{% raw %}rate(vm_persistentqueue_read_duration_seconds_total[5m]) > 0.9{% endraw %}'
        for: 15m
        labels:
          severity: warning
        annotations:
          # dashboard: "http://localhost:3000/d/G7Z9GzMGz?viewPanel=99&var-instance={{ $labels.instance }}"
          summary: '{% raw %}Persistent queue reads for instance {{ $labels.instance }} are saturated{% endraw %}'
          description: '{% raw %}Persistent queue reads for vmagent \"{{ $labels.job }}\" (instance {{ $labels.instance }}) are saturated by more than 90% and vmagent wont be able to keep up with reading data from the disk. In this case, consider to decrease load on the vmagent or improve the disk throughput.{% endraw %}'
      - alert: SeriesLimitHourReached
        expr: '{% raw %}(vmagent_hourly_series_limit_current_series / vmagent_hourly_series_limit_max_series) > 0.9{% endraw %}'
        labels:
          severity: critical
        annotations:
          # dashboard: "http://localhost:3000/d/G7Z9GzMGz?viewPanel=88&var-instance={{ $labels.instance }}"
          summary: '{% raw %}Instance {{ $labels.instance }} reached 90% of the limit{% endraw %}'
          description: '{% raw %}Max series limit set via -remoteWrite.maxHourlySeries flag is close to reaching the max value. Then samples for new time series will be dropped instead of sending them to remote storage systems.{% endraw %}'
      - alert: SeriesLimitDayReached
        expr: '{% raw %}(vmagent_daily_series_limit_current_series / vmagent_daily_series_limit_max_series) > 0.9{% endraw %}'
        labels:
          severity: critical
        annotations:
          # dashboard: "http://localhost:3000/d/G7Z9GzMGz?viewPanel=90&var-instance={{ $labels.instance }}"
          summary: '{% raw %}Instance {{ $labels.instance }} reached 90% of the limit{% endraw %}'
          description: '{% raw %}Max series limit set via -remoteWrite.maxDailySeries flag is close to reaching the max value. Then samples for new time series will be dropped instead of sending them to remote storage systems.{% endraw %}'
      - alert: ConfigurationReloadFailure
        expr: '{% raw %}vm_promscrape_config_last_reload_successful != 1 or vmagent_relabel_config_last_reload_successful != 1{% endraw %}'
        labels:
          severity: warning
        annotations:
          summary: '{% raw %}Configuration reload failed for vmagent instance {{ $labels.instance }}{% endraw %}'
          description: '{% raw %}Configuration hot-reload failed for vmagent on instance {{ $labels.instance }}. Check vmagents logs for detailed error message.{% endraw %}'

# Contains default list of alerts for vmalert service.
# The alerts below are just recommendations and may require some updates
# and threshold calibration according to every specific setup.
vmalert_docker_alerts_vmalert:
  - name: vmalert
    interval: 30s
    rules:
      - alert: ConfigurationReloadFailure
        expr: '{% raw %}vmalert_config_last_reload_successful != 1{% endraw %}'
        labels:
          severity: warning
        annotations:
          summary: '{% raw %}Configuration reload failed for vmalert instance {{ $labels.instance }}{% endraw %}'
          description: '{% raw %}Configuration hot-reload failed for vmalert on instance {{ $labels.instance }}. Check vmalerts logs for detailed error message.{% endraw %}'
      - alert: AlertingRulesError
        expr: '{% raw %}sum(increase(vmalert_alerting_rules_errors_total[5m])) without(alertname, id) > 0{% endraw %}'
        for: 5m
        labels:
          severity: warning
        annotations:
          # dashboard: "http://localhost:3000/d/LzldHAVnz?viewPanel=13&var-instance={{ $labels.instance }}&var-group={{ $labels.group }}"
          summary: '{% raw %}Alerting rules are failing for vmalert instance {{ $labels.instance }}{% endraw %}'
          description: '{% raw %}Alerting rules execution is failing for group \"{{ $labels.group }}\". Check vmalerts logs for detailed error message.{% endraw %}'
      - alert: RecordingRulesError
        expr: '{% raw %}sum(increase(vmalert_recording_rules_errors_total[5m])) without(recording, id) > 0{% endraw %}'
        for: 5m
        labels:
          severity: warning
        annotations:
          # dashboard: "http://localhost:3000/d/LzldHAVnz?viewPanel=30&var-instance={{ $labels.instance }}&var-group={{ $labels.group }}"
          summary: '{% raw %}Recording rules are failing for vmalert instance {{ $labels.instance }}{% endraw %}'
          description: '{% raw %}Recording rules execution is failing for group \"{{ $labels.group }}\". Check vmalerts logs for detailed error message.{% endraw %}'
      - alert: RecordingRulesNoData
        expr: '{% raw %}sum(vmalert_recording_rules_last_evaluation_samples) without(recording, id) < 1{% endraw %}'
        for: 30m
        labels:
          severity: info
        annotations:
          # dashboard: "http://localhost:3000/d/LzldHAVnz?viewPanel=33&var-group={{ $labels.group }}"
          summary: '{% raw %}Recording rule {{ $labels.recording }} ({ $labels.group }}) produces no data{% endraw %}'
          description: '{% raw %}Recording rule \"{{ $labels.recording }}\" from group \"{{ $labels.group }}\" produces 0 samples over the last 30min. It might be caused by a misconfiguration or incorrect query expression.{% endraw %}'
      - alert: TooManyMissedIterations
        expr: '{% raw %}increase(vmalert_iteration_missed_total[5m]) > 0{% endraw %}'
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: '{% raw %}vmalert instance {{ $labels.instance }} is missing rules evaluations{% endraw %}'
          description: '{% raw %}vmalert instance {{ $labels.instance }} is missing rules evaluations for group \"{{ $labels.group }}\". The group evaluation time takes longer than the configured evaluation interval. This may result in missed alerting notifications or recording rules samples. Try increasing evaluation interval or concurrency of group \"{{ $labels.group }}\". See https://docs.victoriametrics.com/vmalert.html#groups. If rule expressions are taking longer than expected, please see https://docs.victoriametrics.com/Troubleshooting.html#slow-queries.{% endraw %}'
      - alert: RemoteWriteErrors
        expr: '{% raw %}increase(vmalert_remotewrite_errors_total[5m]) > 0{% endraw %}'
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: '{% raw %}vmalert instance {{ $labels.instance }} is failing to push metrics to remote write URL{% endraw %}'
          description: '{% raw %}vmalert instance {{ $labels.instance }} is failing to push metrics generated via alerting or recording rules to the configured remote write URL. Check vmalerts logs for detailed error message.{% endraw %}'
      - alert: AlertmanagerErrors
        expr: '{% raw %}increase(vmalert_alerts_send_errors_total[5m]) > 0{% endraw %}'
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: '{% raw %}vmalert instance {{ $labels.instance }} is failing to send notifications to Alertmanager{% endraw %}'
          description: '{% raw %}vmalert instance {{ $labels.instance }} is failing to send alert notifications to \"{{ $labels.addr }}\". Check vmalerts logs for detailed error message.{% endraw %}'

# Contains default list of alerts for various VM components.
# The following alerts are recommended for use for any VM installation.
# The alerts below are just recommendations and may require some updates
# and threshold calibration according to every specific setup.
vmalert_docker_alerts_vmhealth:
  - name: vm-health
    # note the `job` filter and update accordingly to your setup
    rules:
      - alert: TooManyRestarts
        expr: '{% raw %}changes(process_start_time_seconds{job=~".*(victoriametrics|vmselect|vminsert|vmstorage|vmagent|vmalert|vmsingle|vmalertmanager|vmauth).*"}[15m]) > 2{% endraw %}'
        labels:
          severity: critical
        annotations:
          summary: '{% raw %}{{ $labels.job }} too many restarts (instance {{ $labels.instance }}){% endraw %}'
          description: '{% raw %}Job {{ $labels.job }} (instance {{ $labels.instance }}) has restarted more than twice in the last 15 minutes. It might be crashlooping.{% endraw %}'
      - alert: ServiceDown
        expr: '{% raw %}up{job=~".*(victoriametrics|vmselect|vminsert|vmstorage|vmagent|vmalert|vmsingle|vmalertmanager|vmauth).*"} == 0{% endraw %}'
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: '{% raw %}Service {{ $labels.job }} is down on {{ $labels.instance }}{% endraw %}'
          description: '{% raw %}{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 2 minutes.{% endraw %}'
      - alert: ProcessNearFDLimits
        expr: '{% raw %}(process_max_fds - process_open_fds) < 100{% endraw %}'
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: '{% raw %}Number of free file descriptors is less than 100 for \"{{ $labels.job }}\"(\"{{ $labels.instance }}\") for the last 5m{% endraw %}'
          description: '{% raw %}Exhausting OS file descriptors limit can cause severe degradation of the process. Consider to increase the limit as fast as possible.{% endraw %}'
      - alert: TooHighMemoryUsage
        expr: '{% raw %}(min_over_time(process_resident_memory_anon_bytes[10m]) / vm_available_memory_bytes) > 0.8{% endraw %}'
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: '{% raw %}It is more than 80% of memory used by \"{{ $labels.job }}\"(\"{{ $labels.instance }}\"){% endraw %}'
          description: '{% raw %}Too high memory usage may result into multiple issues such as OOMs or degraded performance. Consider to either increase available memory or decrease the load on the process.{% endraw %}'
      - alert: TooHighCPUUsage
        expr: '{% raw %}rate(process_cpu_seconds_total[5m]) / process_cpu_cores_available > 0.9{% endraw %}'
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: '{% raw %}More than 90% of CPU is used by \"{{ $labels.job }}\"(\"{{ $labels.instance }}\") during the last 5m{% endraw %}'
          description: '{% raw %}Too high CPU usage may be a sign of insufficient resources and make process unstable. Consider to either increase available CPU resources or decrease the load on the process.{% endraw %}'
      - alert: TooManyLogs
        expr: '{% raw %}sum(increase(vm_log_messages_total{level="error"}[5m])) without (app_version, location) > 0{% endraw %}'
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: '{% raw %}Too many logs printed for job \"{{ $labels.job }}\" ({{ $labels.instance }}){% endraw %}'
          description: '{% raw %}Logging rate for job \"{{ $labels.job }}\" ({{ $labels.instance }}) is {{ $value }} for last 15m.\n Worth to check logs for specific error messages.{% endraw %}'
      - alert: TooManyTSIDMisses
        expr: '{% raw %}rate(vm_missing_tsids_for_metric_id_total[5m]) > 0{% endraw %}'
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: '{% raw %}Too many TSID misses for job \"{{ $labels.job }}\" ({{ $labels.instance }}){% endraw %}'
          description: '{% raw %}The rate of TSID misses during query lookups is too high for \"{{ $labels.job }}\" ({{ $labels.instance }}).\n Make sure you re running VictoriaMetrics of v1.85.3 or higher.\n Related issue https://github.com/VictoriaMetrics/VictoriaMetrics/issues/3502{% endraw %}'
      - alert: ConcurrentInsertsHitTheLimit
        expr: '{% raw %}avg_over_time(vm_concurrent_insert_current[1m]) >= vm_concurrent_insert_capacity{% endraw %}'
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: '{% raw %}{{ $labels.job }} on instance {{ $labels.instance }} is constantly hitting concurrent inserts limit{% endraw %}'
          description: '{% raw %}The limit of concurrent inserts on instance {{ $labels.instance }} depends on the number of CPUs.\n Usually, when component constantly hits the limit it is likely the component is overloaded and requires more CPU. In some cases for components like vmagent or vminsert the alert might trigger if there are too many clients making write attempts. If vmagents or vminserts CPU usage and network saturation are at normal level, then it might be worth adjusting `-maxConcurrentInserts` cmd-line flag.{% endraw %}'

# Here zou can add zpur custom alerts
vmalert_docker_alerts_custom: []
```

## [Requirements](#requirements)

- pip packages listed in [requirements.txt](https://github.com/mullholland/ansible-role-vmalert_docker/blob/master/requirements.txt).

## [State of used roles](#state-of-used-roles)

The following roles are used to prepare a system. You can prepare your system in another way.

| Requirement | GitHub | GitLab |
|-------------|--------|--------|
|[mullholland.repository_epel](https://galaxy.ansible.com/mullholland/repository_epel)|[![Build Status GitHub](https://github.com/mullholland/ansible-role-repository_epel/workflows/Ansible%20Molecule/badge.svg)](https://github.com/mullholland/ansible-role-repository_epel/actions)|[![Build Status GitLab](https://gitlab.com/opensourceunicorn/ansible-role-repository_epel/badges/master/pipeline.svg)](https://gitlab.com/opensourceunicorn/ansible-role-repository_epel)|
|[mullholland.docker](https://galaxy.ansible.com/mullholland/docker)|[![Build Status GitHub](https://github.com/mullholland/ansible-role-docker/workflows/Ansible%20Molecule/badge.svg)](https://github.com/mullholland/ansible-role-docker/actions)|[![Build Status GitLab](https://gitlab.com/opensourceunicorn/ansible-role-docker/badges/master/pipeline.svg)](https://gitlab.com/opensourceunicorn/ansible-role-docker)|
|[mullholland.pip](https://galaxy.ansible.com/mullholland/pip)|[![Build Status GitHub](https://github.com/mullholland/ansible-role-pip/workflows/Ansible%20Molecule/badge.svg)](https://github.com/mullholland/ansible-role-pip/actions)|[![Build Status GitLab](https://gitlab.com/opensourceunicorn/ansible-role-pip/badges/master/pipeline.svg)](https://gitlab.com/opensourceunicorn/ansible-role-pip)|

## [Context](#context)

This role is a part of many compatible roles. Have a look at [the documentation of these roles](https://mullholland.net) for further information.

Here is an overview of related roles:
![dependencies](https://raw.githubusercontent.com/mullholland/ansible-role-vmalert_docker/png/requirements.png "Dependencies")

## [Compatibility](#compatibility)

This role has been tested on these [container images](https://hub.docker.com/u/mullholland):

|container|tags|
|---------|----|
|[EL](https://hub.docker.com/r/mullholland/enterpriselinux)|all|
|[Fedora](https://hub.docker.com/r/mullholland/fedora/)|38, 39|
|[Ubuntu](https://hub.docker.com/r/mullholland/ubuntu)|all|
|[Debian](https://hub.docker.com/r/mullholland/debian)|all|

The minimum version of Ansible required is 2.10, tests have been done to:

- The previous version.
- The current version.
- The development version.

If you find issues, please register them in [GitHub](https://github.com/mullholland/ansible-role-vmalert_docker/issues).

## [License](#license)

[MIT](https://github.com/mullholland/ansible-role-vmalert_docker/blob/master/LICENSE).

## [Author Information](#author-information)

[mullholland](https://mullholland.net)
