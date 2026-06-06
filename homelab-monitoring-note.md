# ホームラボを監視しよう。Prometheus + Grafanaで始めるお手軽サーバー監視入門

## やったこと

自宅サーバー、いわゆるホームラボの監視環境を Prometheus + Grafana で構築した。

構成はコンテナを使わず、各コンポーネントを systemd サービスとして動かす形にした。自宅サーバー用途では、Docker や Podman に寄せすぎるより、OS の標準的な service 管理に乗せた方が見通しが良いと判断した。

今回構築した構成は以下。

```text
母艦サーバー
  - Grafana
  - Prometheus
  - node_exporter

子機 Raspberry Pi
  - node_exporter
```

最終的な待ち受けポートはこうなった。

```text
Grafana       : 3000
Cockpit       : 9090
Prometheus    : 127.0.0.1:9091
node_exporter : 9100
```

RHEL 系では Cockpit が `9090` を使うため、Prometheus は `9091` に変更した。

## Prometheus と Grafana の役割

Prometheus はメトリクスを収集するための仕組み。

監視対象に exporter を置き、Prometheus が定期的に HTTP でメトリクスを取得する。今回は Linux サーバーの CPU、メモリ、ディスク、ネットワークなどを見るために node_exporter を使った。

Grafana は Prometheus に保存されたメトリクスを見やすく可視化するためのダッシュボード。

つまり役割分担はこう。

```text
node_exporter -> メトリクスを公開
Prometheus    -> メトリクスを収集
Grafana       -> メトリクスを可視化
```

## Ansible で母艦サーバーを構築

母艦サーバーには Ansible で以下を導入した。

- Prometheus
- node_exporter
- Grafana
- Grafana の Prometheus datasource
- firewalld の `3000/tcp` 開放
- SELinux 対応

Grafana は LAN から見たいので `3000/tcp` を開けた。

一方で Prometheus は Grafana から内部的に参照できればよいので、外部公開せず `127.0.0.1:9091` に閉じた。

```yaml
prometheus_listen_address: "127.0.0.1:9091"
grafana_prometheus_datasource_url: http://127.0.0.1:9091
monitoring_firewalld_ports:
  - 3000/tcp
```

実行は以下。

```bash
ansible-playbook monitoring.yml
```

## Grafana の初期ログイン

初期ログインは以下。

```text
user: admin
password: admin
```

ただし、そのままだと弱いので Ansible 側で `grafana_admin_password` を上書きする。

```yaml
grafana_admin_password: "任意の強いパスワード"
```

## Cockpit と Prometheus のポート衝突

最初は Prometheus を `9090` で動かそうとしたが、RHEL では Cockpit が `9090` を使っていた。

そのため Prometheus を `9091` に変更した。

確認コマンド。

```bash
systemctl status prometheus --no-pager
curl -s http://127.0.0.1:9091/-/ready
```

`Prometheus Server is Ready.` が返れば起動している。

## SELinux で Grafana から Prometheus に接続できない問題

Grafana の datasource で Prometheus を指定したところ、以下のようなエラーが出た。

```text
dial tcp 127.0.0.1:9091: connect: permission denied
```

Prometheus 自体は起動しており、curl では疎通できていた。

つまり原因はネットワークではなく SELinux。

Grafana が Prometheus のローカルポートへ接続できるように、SELinux の boolean とローカル policy を調整した。

```bash
sudo setsebool -P grafana_can_tcp_connect_prometheus_port on
```

今回の環境では `prometheus_port_t` が使えなかったため、ローカル SELinux type を作って `9091/tcp` に割り当てた。

```cil
(type prometheus_grafana_port_t)
(typeattributeset port_type (prometheus_grafana_port_t))
(allow grafana_t prometheus_grafana_port_t (tcp_socket (name_connect)))
```

その後、Grafana の datasource を以下に設定した。

```text
http://127.0.0.1:9091
```

`Save & test` が通れば OK。

## Grafana ダッシュボード

Grafana では Node Exporter Full を import した。

```text
Dashboard ID: 1860
Name: Node Exporter Full
```

CPU、メモリ、ディスク、ネットワーク、filesystem などがまとまって見られる定番 dashboard。

Prometheus datasource として `Prometheus` を選択すると、母艦サーバーのメトリクスが表示された。

また、Grafana の Home Dashboard に設定すれば、ログイン後すぐこの監視画面を表示できる。

## Raspberry Pi を子機として追加

次に Raspberry Pi を監視対象として追加した。

子機には Prometheus や Grafana は入れず、node_exporter だけを入れる。

Raspberry Pi 側で node_exporter を起動し、母艦の Prometheus から `192.168.1.151:9100` を scrape する。

Prometheus の設定は以下のようにした。

```yaml
scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets:
          - localhost:9091

  - job_name: node
    static_configs:
      - targets:
          - localhost:9100
          - 192.168.1.151:9100
```

ポイントは Raspberry Pi も `job_name: node` に入れること。

最初は誤って `job_name: prometheus` に入れてしまったが、Node Exporter Full で自然に扱うなら `node` に入れるのが正しい。

Prometheus の target API で確認した。

```bash
curl -s http://127.0.0.1:9091/api/v1/targets
```

最終的に以下のように `health: up` になった。

```text
localhost:9100       job=node  health=up
192.168.1.151:9100   job=node  health=up
localhost:9091       job=prometheus  health=up
```

Grafana の Node Exporter Full でも `Instance` に以下が表示されるようになった。

```text
localhost:9100
192.168.1.151:9100
```

これで母艦サーバーと Raspberry Pi を同じ dashboard で切り替えて監視できる。

## まとめ

Prometheus + Grafana + node_exporter を使うと、ホームラボの基本的な監視はかなり手軽に始められる。

今回のポイントは以下。

- Grafana は `3000/tcp` を LAN に公開
- RHEL の Cockpit と被るため Prometheus は `9091` に変更
- Prometheus は外部公開せず `127.0.0.1` に閉じる
- SELinux で Grafana から Prometheus への接続を許可する
- 子機には node_exporter だけ入れる
- Prometheus の `job_name: node` に母艦と子機の exporter を並べる

次にやるなら、Grafana Alerting で以下を通知したい。

- ディスク使用率 85% 超え
- メモリ逼迫
- node_exporter down
- Prometheus target down

ホームラボでも、監視があるだけで障害の原因調査がかなり楽になる。
