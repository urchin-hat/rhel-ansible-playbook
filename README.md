# rhel-ansible-playbook
An Ansible playbook for RHEL in my environment

## Monitoring

Prometheus, node_exporter, and Grafana can be installed without containers.

```bash
ansible-playbook monitoring.yml
```

Default endpoints:

- Grafana: `http://<server>:3000`
- Prometheus: `http://localhost:9090`
- node_exporter: `http://localhost:9100/metrics`

Only Grafana is exposed through firewalld by default. Prometheus and
node_exporter bind to `127.0.0.1`.

Override sensitive or host-specific values in `group_vars/all.yml` or host vars.

```yaml
grafana_admin_password: "change-this-password"
prometheus_scrape_targets:
  - job_name: prometheus
    targets:
      - localhost:9090
  - job_name: node
    targets:
      - localhost:9100
```
