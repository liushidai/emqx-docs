# Tuning guide

Since 4.2 EMQX had been stress-tested with 1.3 million on an 8-core, 32G memory CentOS server.

This guide includes in general tuning suggestions for one EMQX broker to serve about 1 million clients.

## Turn off swap

Linux swap partitions may cause nondeterministic memory latency to Erlang virtual machine,
which in turn significantly affects the system stability.
It is recommended to turn off swap permanently.

To turn off swap immediately, execute command `sudo swapoff -a`.
To turn off swap permanently, comment out the `swap` line in `/etc/fstab` and reboot the host

## Linux Kernel Tuning

The system-wide limit on max opened file handles:

```
# 2 millions system-wide
sysctl -w fs.file-max=2097152
sysctl -w fs.nr_open=2097152
echo 2097152 > /proc/sys/fs/nr_open
```

The limit on opened file handles for current session:

```
ulimit -n 2097152
```

### /etc/sysctl.conf

Persist 'fs.file-max' configuration to `/etc/sysctl.conf`:

```
fs.file-max = 2097152
```

Set the maximum number of file handles for the service in `/etc/systemd/system.conf`:

```
DefaultLimitNOFILE=2097152
```

### emqx.service

Set the maximum number of file handles for emqx service in e.g. one of below paths depending
on which linux distribution is in use.

- `/usr/lib/systemd/system/emqx.service`
- `/lib/systemd/system/emqx.service`

```
LimitNOFILE=2097152
```

### /etc/security/limits.conf

Persist the maximum number of opened file handles for users in `/etc/security/limits.conf`:

```
*      soft   nofile      2097152
*      hard   nofile      2097152
```

## TCP Network Tuning

Increase number of incoming connections backlog:

```
sysctl -w net.core.somaxconn=32768
sysctl -w net.ipv4.tcp_max_syn_backlog=16384
sysctl -w net.core.netdev_max_backlog=16384
```

Local port range

```
sysctl -w net.ipv4.ip_local_port_range='1000 65535'
```

TCP Socket read/write buffer:

```
sysctl -w net.core.rmem_default=262144
sysctl -w net.core.wmem_default=262144
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216
sysctl -w net.core.optmem_max=16777216

#sysctl -w net.ipv4.tcp_mem='16777216 16777216 16777216'
sysctl -w net.ipv4.tcp_rmem='1024 4096 16777216'
sysctl -w net.ipv4.tcp_wmem='1024 4096 16777216'
```

TCP connection tracking:

```
sysctl -w net.netfilter.nf_conntrack_max=1000000
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30
```

TIME-WAIT Bucket Pool, Recycling and Reuse:

```
sysctl -w net.ipv4.tcp_max_tw_buckets=1048576

# Enabling following option is not recommended. It could cause connection reset under NAT
# sysctl -w net.ipv4.tcp_tw_recycle=1
# sysctl -w net.ipv4.tcp_tw_reuse=1
```

Timeout for FIN-WAIT-2 Sockets:

```
sysctl -w net.ipv4.tcp_fin_timeout=15
```

## Erlang VM Tuning


Tuning and optimize the Erlang VM in etc/emqx.conf file


```bash
## Erlang Process Limit
node.process_limit = 2097152

## Sets the maximum number of simultaneously existing ports for this system
node.max_ports = 2097152
```

## When running in docker

Usually you should tune the linux docker host by following the above guide.

If you want to tune linux kernel by docker, you must ensure your docker is latest version (>=1.12).

Here is an example to show how it looks.

```
docker run -d --name emqx -p 18083:18083 -p 1883:1883 \
    --sysctl fs.file-max=2097152 \
    --sysctl fs.nr_open=2097152 \
    --sysctl net.core.somaxconn=32768 \
    --sysctl net.ipv4.tcp_max_syn_backlog=16384 \
    --sysctl net.core.netdev_max_backlog=16384 \
    --sysctl net.ipv4.ip_local_port_range='1000 65535' \
    --sysctl net.core.rmem_default=262144 \
    --sysctl net.core.wmem_default=262144 \
    --sysctl net.core.rmem_max=16777216 \
    --sysctl net.core.wmem_max=16777216 \
    --sysctl net.core.optmem_max=16777216 \
    --sysctl net.ipv4.tcp_rmem='1024 4096 16777216' \
    --sysctl net.ipv4.tcp_wmem='1024 4096 16777216\ \
    --sysctl net.ipv4.tcp_max_tw_buckets=1048576 \
    --sysctl net.ipv4.tcp_fin_timeout=15 \
    emqx/emqx:latest
```

::: REMEMBER
The best practice is NOT to run docker `--privileged` and NOT to mount system volumes to the container for kernel tuning.
:::

## EMQX Broker Tuning

Tune the acceptor pool, max_clients limit and socket options.
{% emqxce %}
Find listeners config in `etc/emqx.conf`
{% endemqxce %}
{% emqxee %}
Find listeners config in `etc/listeners.conf`
{% endemqxee %}

```bash
## TCP Listener
listener.tcp.external = 0.0.0.0:1883
listener.tcp.external.acceptors = 64
listener.tcp.external.max_connections = 1024000
```

## Client Machine Tuning

 Tune the client machine to benchmark emqttd broker:
```
sysctl -w net.ipv4.ip_local_port_range="500 65535"
echo 1000000 > /proc/sys/fs/nr_open
ulimit -n 100000
```
### emqtt_bench

 Test tool for concurrent connections:  <http://github.com/emqx/emqtt_bench>
