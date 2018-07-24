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

Whether or not the SignalFlow program uses any form of aggregation, this driver presently returns a single value for a single metric, named `perf` by default.  This value is computed from the time series data returned from a single SignalFlow API query as follows:

* space aggregation:  for a given time, aggregate all of the data values to produce a single value using one of these methods:  avg, max, min, sum (if there is just one time series pointlist, these are all the same)
* time aggregation:  aggregate all of the space aggregates to produce a single value (e.g., perf metric) using one of these methods:  avg, max, min, sum

## servo configuration

A servo which uses this measure driver requires a SignalFx API token to exist either as the contents of the file /etc/optune-sfx-auth/api_key or as the value of the environment variable OPTUNE_SFX_API_KEY.

The file method is suitable for use with Kubernetes secrets.  This file can be automatically created on a Kubernetes servo using a secret mounted as `/etc/optune-signalfx-auth`.  See the following example.

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

In addition to the standard `warmup` and `duration` configuration, the driver measurement control specification provides for configuring the SignalFlow program, and the method(s) for computing a single `perf` result, using the free-from `userdata` section of the operator override descriptor (e.g., an app-desc.yaml provided to the server).  For example:

```
measurement:
  control:
    warmup:    30
    duration:  130
    userdata:
      program:  'data('pod_network_transmit_bytes_total',filter('kubernetes_namespace','abc')).mean().publish()'
      time_aggr:   avg       # compute perf metric value as the average of
                             # space aggregates
      space_aggr:  sum       # compute per-time value as the sum of all
                             # values for that time
      pre_cmd_async:  'ab -c 10 -rkl -t 1000 http://c4:8080/'
```

* `warmup`:  period after adjustment when a measurement is not taken (sleep).  Default 0 seconds.
* `duration`:  period of measurement.  Default 120 seconds.
* `program`: a SignalFlow program (e.g. used to query time-series metrics).  Required. 
* `time_aggr`:  aggregation method for space aggregate values (used to create a single `perf` metric).  One of avg|max|min|sum.  Default `avg`.
* `space_aggr`:  aggregation method for all values at a given time.  One of avg|max|min|sum.  Default `avg`.
* `pre_cmd_async`:  Bash shell command to execute prior to warmup.  This optional command may be a string or a list.  This command is executed asynchronously with stdout/stderr directed to /dev/null.  If the process is still running after measurement, it is terminated.  This command is suitable for generating load during measurement, typically for testing purposes, as in the example above.
* `pre_cmd`:  Bash shell command to execute prior to warmup.  This optional command may be a string or a list.
* `post_cmd`:  Bash shell command to execute after measurement.  This optional command may be a string or a list.
* `stream_endpoint`:  endpoint for SignalFx stream API.  Default `https://stream.signalfx.com`.

**TBD** implementation required...
* `metric_name`:  metric name.  Default `perf`.
* `metric_unit`:  metric unit, returned by the `describe` command.  Default '' (empty string).
