# Monitoring and Alerting

## Monitoring

We use [Prometheus](https://prometheus.io/) for monitoring and alerting.
Prometheus is an open-source systems monitoring and alerting toolkit originally
built at SoundCloud. It has a large ecosystem of integrations and is widely
used in the industry. Prometheus collects metrics from configured targets at
specified intervals, evaluates rule expressions, and can trigger alerts if
certain conditions are met. It also provides a powerful query language (PromQL)
for querying and aggregating metrics data.

### Accessing the Prometheus Server

It can be useful to interact with the cluster's prometheus server while
developing dashboards in grafana. You will need to forward a local port
to the prometheus server's pod.

### Using the standard port

Listen on port 9090 locally, forwarding to the prometheus server's port
`9090`.

``` bash
kubectl -n support port-forward deployment/support-prometheus-server 9090
```

then visit http://localhost:9090.

### Using an alternative port

Listen on port 8000 locally, forwarding to the prometheus server's port `9090`.

``` bash
kubectl -n support port-forward deployment/support-prometheus-server 8000:9090
```

then visit http://localhost:8000

### Grafana

[Grafana](https://grafana.com/) is used to visualize the metrics collected by
Prometheus. Grafana is an open-source analytics and monitoring solution that
integrates with various data sources, including Prometheus. It provides a rich
set of visualization options and allows users to create custom dashboards for
monitoring their systems.

Our Grafana instance is hosted at
[https://grafana.cal-icor.org](https://grafana.cal-icor.org). You can log in
using your GitHub credentials if you're part of the `Grafana Access` team. If
you need access, please contact the CAL-ICOR team by creating a Github issue in
the [cal-icor/cal-icor-hubs](https://github.com/cal-icor/cal-icor-hubs/issues)
repository.

Upstream documentation is found
[here](https://jupyterhub-grafana.readthedocs.io/en/latest/index.html)

## Alerting

We have set up alerting rules in Grafana and GCP Monitoring to notify the
Cal-ICOR team of any issues with the JupyterHub deployment. These alerts are
based on the metrics collected by Prometheus and are designed to help us
proactively monitor the health and performance of the system.

The alerts are configured to trigger notifications via
[PagerDuty](https://cal-icor.pagerduty.com) and email, ensuring that the team
is promptly informed of any critical issues that may arise. The alerts cover
various aspects of the JupyterHub deployment, including resource usage, system
performance, and user activity.
