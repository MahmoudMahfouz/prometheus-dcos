# [Prometheus](https://prometheus.io/) on [Mesosphere DC/OS](https://dcos.io/)

## Intro
This runs Prometheus on DC/OS. `server.json` contains the service definition for Prometheus itself. `node_exporter.json` contains the service definition for node_exporter. I'm running node_exporter inside a Mesos (cgroups) container so that it sees all of the hosts filesystems without any need for priviliges or translation.

## Why file_sd based discovery?
Prometheus supports DNS based service discovery. Given a Mesos-DNS SRV record like `_node-exporter.prometheus._tcp.marathon.mesos` it will find the list of node_exporter nodes and poll them. However it'll result in instance names like
```
node-exporter.prometheus-6ms1y-s1.marathon.mesos:14181
node-exporter.prometheus-54eio-s0.marathon.mesos:12227
node-exporter.prometheus-1e1ow-s2.marathon.mesos:31798
```
which is not very useful. Also the Mesos scheduler will assign a random port resource.

So after a [discussion on the mailing list](https://groups.google.com/forum/#!topic/prometheus-developers/ydww-vzG0IE) it turned out that Prometheus can't relabel the instance with the node's IP address since name resolution happens after relabeling. It was suggested to use the file_sd based discovery method instead. This is what the `srv-lookup` helper is for. It performs the same SRV and A record lookup and instead of the hostname writes the node's IP addres into the targets file. There's also relabeling taking place to replace the random port number with the node_exporter standard port 9100 so that when a node_exporter is restarted on a different port it's data is still associated with the same node.

## Environment Variables
| Variable | Function | Example |
|----------|----------|-------|
|`NODE_EXPORTER_SRV` | Mesos-DNS SRV record of the node_exporter | `NODE_EXPORTER_SRV=_node-exporter.prometheus._tcp.marathon.mesos`|
|`SRV_REFRESH_INTERVAL` | How often should we update the targets JSON | `SRV_REFRESH_INTERVAL=60`|
|`ALERT_MANAGER_URI` | AlertManager URL | `ALERT_MANAGER_URI=http://prometheusalertmanager.marathon.l4lb.thisdcos.directory:9093`|

## Building the SRV lookup helper
To run the srv-lookup helper tool inside the minimal prom/prometheus Docker container I statically linked it. To do so yourself install [musl libc](http://www.musl-libc.org/) and compile using:
```
$ CC=/usr/local/musl/bin/musl-gcc go build --ldflags '-linkmode external -extldflags "-static"' srv-lookup.go
```

## Bugs
All this was hacked up in an afternoon. Surely there's bugs. If you find any submit a PR or open an issue.

## TODO
- perform A lookups in parallel instead of looping over all hosts sequentially