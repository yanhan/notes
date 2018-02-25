# About

Some notes on statsd

## Manually send artificial data to statsd

Some python code that does it:

```
import socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.sendto("custom_metric:60|g|#shell", ("localhost", 8125))
```

This assumes that there is a statsd server running on port 8125 locally.

The 3rd line of the code should be changed accordingly depending on the metric you wish to send. The format of the message is as follows:

```
metricname:value_of_count|c|#tag1:val1,tag2:val2
```

So for a counter named `odin.http.count` that we wish to increment by 5 and has tags `system:zero` and `sky:blue`, the message will be:

```
odin.http.count:5|c|#system:zero,sky:blue
```

For more information, see https://docs.datadoghq.com/developers/dogstatsd/#metrics-1
