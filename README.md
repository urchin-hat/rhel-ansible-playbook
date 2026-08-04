# rhel-ansible-playbook
An Ansible playbook for RHEL in my environment

## k3s on Podman

Install a single-node k3s server on the `local` inventory group. k3s runs in a
privileged, rootful Podman container managed by systemd; no Docker daemon is
installed.

```bash
ansible-playbook k3s.yml
sudo podman exec k3s kubectl get nodes
```

The Kubernetes API listens on port `6443`. The role persists k3s state below
`/var/lib/rancher/k3s` and writes kubeconfig to
`/etc/rancher/k3s/k3s.yaml`. Override settings in `group_vars/local.yml`, for
example:

```yaml
k3s_podman_image: docker.io/rancher/k3s:v1.36.1-k3s1
k3s_podman_tls_sans:
  - 192.168.1.100
k3s_podman_token: "{{ vault_k3s_token }}"
```

The outer container is run by Podman, while k3s continues to use its bundled
containerd as the Kubernetes CRI inside that container. Running k3s directly
with Podman as its CRI is not supported.

## Monitoring

Prometheus, node_exporter, and Grafana can be installed without containers.

```bash
ansible-playbook monitoring.yml
```

Default endpoints:

- Grafana: `http://<server>:3000`
- Prometheus: `http://localhost:9091`
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

Grafana connects to Prometheus through `http://127.0.0.1:9091` by default.
Port `9090` is left available for Cockpit on RHEL hosts.
On SELinux-enabled RHEL hosts, the role also enables
`httpd_can_network_connect` so Grafana can query local data sources.
