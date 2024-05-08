# Learn Loki

## Install

```

curl -O -L "https://github.com/grafana/loki/releases/download/v2.9.8/loki-linux-amd64.zip"
dnf install unzip
unzip loki-linux-amd64.zip

chmod a+x loki-linux-amd64
cp loki-linux-amd64 /usr/local/bin/loki

export PATH=$PATH:/usr/local/bin/

mkdir /etc/loki
cd /etc/loki


```

vi /etc/loki/config-loki.yml
```
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  path_prefix: /tmp/loki
  storage:
    filesystem:
      chunks_directory: /tmp/loki/chunks
      rules_directory: /tmp/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093
```
This default configuration was copied from: https://raw.githubusercontent.com/grafana/loki/master/cmd/loki/loki-local-config.yaml  
There may be changes to this config depending on any future updates to Loki.


## Create the loki service

```
sudo useradd --system loki
```

vi /etc/systemd/system/loki.service
```
[Unit]
Description=Loki service
After=network.target

[Service]
Type=simple
User=loki
ExecStart=/usr/local/bin/loki-linux-amd64 -config.file /etc/loki/config-loki.yml

[Install]
WantedBy=multi-user.target
```


```
systemctl daemon-reload

systemctl start loki
systemctl enable loki
systemctl status loki
```


## Configure Firewall

If you only want localhost to be able to connect, then type
```
iptables -A INPUT -p tcp -s localhost --dport 3100 -j ACCEPT
iptables -A INPUT -p tcp --dport 3100 -j DROP
iptables -L

curl 127.0.0.1:3100/metrics
```


Also, Loki exposes port 9096 for gRPC communications.
```
iptables -A INPUT -p tcp -s <your grafana servers domain name or ip address> --dport 9096 -j ACCEPT
iptables -A INPUT -p tcp -s localhost --dport 9096 -j ACCEPT
iptables -A INPUT -p tcp --dport 9096 -j DROP
iptables -L
```

Warning!
```
iptables settings will be lost in case of system reboot. You will need to reapply them manually,

or

install iptables-persistent

sudo apt install iptables-persistent
This will save your settings into two files called,

/etc/iptables/rules.v4
/etc/iptables/rules.v6

Any changes you make to the iptables configuration won't be auto saved to these persistent files, so if you want to update these files with any changes, then use the commands,

iptables-save > /etc/iptables/rules.v4
iptables-save > /etc/iptables/rules.v6
```

## Loki API

```
$ curl -S -H "Content-Type: application/json" -XPOST -s http://localhost:3100/loki/api/v1/push --data-raw '{"streams": [{ "stream": { "app": "app1" }, "values": [ [ "1653855518000000000", "random log line" ] ] }]}'

$ curl http://localhost:3100/loki/api/v1/labels
{"status":"success","data":["__name__","app"]}

As you can see in the second curl there is now a new label named "app". We can explore possible values for this label with the following curl.

$ curl http://localhost:3100/loki/api/v1/label/app/values
{"status":"success","data":["app1"]}

And finaly see the stream with this label:

$ curl -G -Ss  http://localhost:3100/loki/api/v1/query_range --data-urlencode 'query={app="app1"}' | jq .
{
  "status": "success",
(...)
        "values": [
          [
            "1653855518000000000",
            "random log line"
          ]
        ]
      }
(...)

```



## LogCLI

LogCLI is the command-line interface to Grafana Loki. It facilitates running LogQL queries against a Loki instance.

Install
```
wget https://github.com/grafana/loki/releases/download/v2.9.8/logcli-linux-amd64.zip
unzip logcli-linux-amd64.zip

cp logcli-linux-amd64 /usr/local/bin/logcli
```

Set up command completion
```
eval "$(logcli --completion-script-bash)"
```

Set the env variable
```
export LOKI_ADDR=http://localhost:3100

export LOKI_USERNAME=admin
export LOKI_PASSWORD=admin
```

```
logcli labels

logcli labels job
>https://logs-dev-ops-tools1.grafana.net/api/prom/label/job/values
>loki-ops/consul
>loki-ops/loki-gw

logcli query '{job="loki-ops/consul"}'


```


## Troubleshooting

### Permission Denied, Internal Server Error

If you get any of these errors
- Loki: Internal Server Error. 500. open /tmp/loki/index/index_2697: permission denied
- "failed to flush user" "open /tmp/loki/chunks/...etc : permission denied"
- Loki: Internal Server Error. 500. Internal Server Error

You should check the owner of the folders configured in the storage_config section of the the config-loki.yml to match the name of the user configured in the loki.service script above.

```
chown -R loki:loki /tmp/loki

You may need to restart the Loki service and checking again its status
```













