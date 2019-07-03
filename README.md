# servo-sfx
Optune servo measure driver for SignalFx

Note: this driver requires `measure.py` base class from the Optune servo core. It can be copied or symlinked here as part of packaging.  A servo using this driver requires the following 3rd party Python3 packages:  `signalfx` (required by this driver) and `requests` (required by the servo core).  When building a servo image, install these packages using pip3, e.g.:  `pip3 install signalfx requests`

## overview

SignalFx metric queries are made using the SignalFlow API `execute` endpoint (<https://developers.signalfx.com/reference#signalflow-execute>).  The data for this query includes the SignalFlow program to execute to obtain time series data.  For example, the program:
```
data('pod_network_transmit_bytes_total',filter('kubernetes_namespace','abc')).publish()
```
returns a set of time-stamped data sets.  Each time-stamped data set may contain one or more values (e.g., from different time series, in this example one value for each pod in the abc namespace), while the set of such sets cover the time span to be assessed.

Note that the SignalFlow language provides for aggregation, so that the above example may be modified to be:
```
data('pod_network_transmit_bytes_total',filter('kubernetes_namespace','abc')).mean().publish()
```
In this case the `mean()` method is used to aggregate a given set of time-stamped data to a single value, so that the data returned includes one value per time (e.g., aggregated for all pods in the abc namespace).

Please, define parameter `by` for aggregation method, so that we can distinguish timeseries across instances of an application. A few good examples: `.mean(by='container_id')` or `.precentile(95, by='kubernetes_pod_name')`. You can put as many clauses in `by` parameter as a list. Keep in mind they will be concatenated and used as an id, e.g. `container_id:abc   kubernetes_pod_name:abc_xyz` (in the form of `name:value` separated by 3 empty spaces).

## servo configuration

A servo which uses this measure driver requires a SignalFx API token to exist either as the contents of the file /etc/optune-sfx-auth/api_key or as the value of the environment variable OPTUNE_SFX_API_KEY.

The file method is suitable for use with Kubernetes secrets.  This file can be automatically created on a Kubernetes servo using a secret mounted as `/etc/optune-sfx-auth`.  See the following example.

Create a secret in namespace `abc` using kubectl:
```
kubectl -n abc create secret generic optune-sfx-auth \
--from-literal=api_key='<my_value>'
```

Configure the servo deployment YAML descriptor:
```
spec:
  template:
    spec:
      volumes:
      - name: sfx
        secret:
          secretName: optune-sfx-auth   
      containers:
      -name main
        ...
        volumeMounts:
        - name: sfx
          mountPath: '/etc/optune-sfx-auth'
          readOnly: true               
```

## driver configuration

The driver measurement control specification provided by the Optune backend during optimization provides the standard `warmup` and `duration` configuration:

* `warmup`:  period in seconds after adjustment when a measurement is not taken (sleep).  Default `0`.
* `duration`:  period of measurement in seconds.  Default `120`.

All other configuration is provided to this driver through a YAML descriptor `config.yaml` to be provided in the same directory as the driver on the servo itself.  Here is an example of such a descriptor:

```
sfx:
    pre_cmd_async: 'ab -c 10 -rkl -t 1000 http://c4:8080/'
    metrics:
        perf:
            flow_program:  "data('pod_network_transmit_bytes_total',filter('kubernetes_namespace','abc')).mean().publish()"
            unit:          bytes
```

The `config.yaml` descriptor must include at least one named metric.  If the metric name is `perf` the value of this metric will be used by default as the performance measurement during optimization.  Multiple named metrics may also be used to compute a performance measurement using a formula defined in the operator override descriptor.  In addition to the metrics specification, this descriptor supports the following configuration:

* `pre_cmd_async`:  Bash shell command to execute prior to warmup.  This optional command may be a string or a list.  This command is executed asynchronously with stdout/stderr directed to /dev/null.  If the process is still running after measurement, it is terminated.  This command is suitable for generating load during measurement, typically for testing purposes, as in the example above.
* `pre_cmd`:  Bash shell command to execute prior to warmup.  This optional command may be a string or a list.
* `post_cmd`:  Bash shell command to execute after measurement.  This optional command may be a string or a list.
* `pre_cmd_tout`:  optional timeout in seconds for `pre_cmd`.
* `post_cmd_tout`:  optional timeout in seconds for `post_cmd`.
* `stream_endpoint`:  endpoint for SignalFx stream API.  Default `https://stream.signalfx.com`.

Each metric in the metrics section of the `config.yaml` descripor supports the following configuration:

* `flow_program`: a SignalFlow program (e.g. used to query time-series metrics).  Required.
* `flow_immediate`:  boolean indicating whether or not to adjust the measurement stop timestamp so the SignalFlow computation does not wait for future data to become available.  Default `False`.
* `flow_resolution`:  the minimum desired data resolution, in milliseconds.  Default `None` (use the default minimum time resolution)
