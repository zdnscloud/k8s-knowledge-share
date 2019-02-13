version info:
    elasticsearch:v6.3.0
    kibana-oss:6.3.2
    fluentd:v2.2.0

how to install:
    1. create a namespace: kubectl -f namespace.yaml
    2. install elasticsearch: kubectl -f elasticsearch/
    3. install kibana: kubectl -f kibana/
    4. install fluentd: kubectl -f fluentd/
