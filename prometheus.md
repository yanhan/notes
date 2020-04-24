## CPU usage monitoring

More details at: https://www.robustperception.io/understanding-machine-cpu-usage

Use this:
```
100 - (avg by (instance) (irate(node_cpu_seconds_total{job="node_exporter",mode="idle"}[5m])) * 100)
```
