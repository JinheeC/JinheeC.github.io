---
title: "RabbitMQ 데이터를 Influx에 적재하기"
categories: 
- Influx
excerpt: |
  작업 배경은 RabbitMQ에 쌓이는 데이터를 RabbitMQ로 부터 Json 형태로 값을 받아와 InfluxDB 에 적재하고 그 데이터를 Grafana Dashboard 로 뿌려줘야하는 상황이었다. 이 글은 RabbitMQ -> InfluxDB -> Grafana 단계의 RabbitMQ -> InfluxDB 단계만을 설명한다.
feature_text: |
  RabbitMQ 데이터를 Influx에 적재했던 기록
feature_image: "https://picsum.photos/2560/600?random"
---


* table of contents
{:toc .toc}


## 작업 배경
작업 배경은 RabbitMQ에 쌓이는 데이터를 RabbitMQ로 부터 Json 형태로 값을 받아와 InfluxDB 에 적재하고 그 데이터를 Grafana Dashboard 로 뿌려줘야하는 상황이었다. 이 글은 RabbitMQ -> InfluxDB -> Grafana 단계의 RabbitMQ -> InfluxDB 단계만을 설명한다.

이 글은 Influx v.1.4.2 telegraf v.1.5.0 을 기준으로 작성되었다.

## 작업 순서
* 서버 발급
* telegraf, influx 설치
* 데이터 누락 문제 인지
* customizing 작업

## 작업 내용
influxDB 는 다양한 데이터를 읽어오기 위한 플러그인을 많이 제공하고 있다. 현재 기준 지원하고 있는 인풋 플러그인은 아래의 부록: telegraf-input plugins 를 참고하면 된다.

### telegraf, influx 설치
설치방법은 아래와 같다.
명시되지 않은 그 이외의 OS 는 [링크](https://portal.influxdata.com/downloads)를 참고하면된다.
#### InfluxDB 
```
- OS X (mac)
brew update
brew install influxdb 
```

```
- CentOS & RedHat (아래 명령어는 v.1.5.3)
wget https://dl.influxdata.com/influxdb/releases/influxdb-1.5.3.x86_64.rpm
sudo yum localinstall influxdb-1.5.3.x86_64.rpm
```

#### Telegraf
```
- OS X (mac)
brew update
brew install telegraf
```

```
- CentOS & RedHat
wget https://dl.influxdata.com/telegraf/releases/telegraf-1.7.0-1.x86_64.rpm
sudo yum localinstall telegraf-1.7.0-1.x86_64.rpm
```

telegraf 는 설치 후 원하는 input plugin 을 붙이기위해서 config 파일을 설정해서 아래의 명령어로 띄우면 되는데

```
telegraf --config /telegraf.conf
```

config 파일은 아래와 같이 원하는 input, output 을 설정하면 telegraf.conf 파일이 생성된다.

```
telegraf --input-filter <pluginname>[:<pluginname>] --output-filter <outputname>[:<outputname>] config > telegraf.conf
```

이 글의 상황의 경우 input은 rabbitMQ, output 은 influxdb 였기 때문에 아래와 같은 명령어로 config 파일을 만들어 주었다.

```
./telegraf --input-filter amqp_consumer --output-filter influxdb config > telegraf.conf
```

생성된 config 파일에는 본인의 서버 정보를 맞게 작성해 주면 된다. 그리고 위에 적어두었던 telegraf 실행 명령어로 띄워주면 이제 RabbitMQ 의 Json 데이터를 가져와 Influx 에 넣어줄 것이다. 근데 Influx 데이터를 보면 원하는 값이 전부 들어오지 않았을 수 도 있다. 

### 데이터 누락 문제 인지

그 이유는 telegraf 소스를 보면 알 수 있다. telegraf 를 설치했다면 GoPath (/go/src 가 있는 곳)가 있을텐데 이 루트에 존재하는 `/go/src/github.com/influxdata/telegraf/plugins/parsers/json/parser.go` 를 보면 Json을 파싱해서 값을 가져올 때 value 가 Boolean, String인 경우 가져오지 않는 것을 볼 수 있다. 그 부분의 코드는 아래와 같다. FlattenJson 이 파싱할때 호출이되고 이 메소드에서 FullFlattenJson() 을 호출하게되는데 보는것과 같이 FullFlattenJSON(fieldname, v, false, false) 으로 String, bool 값은 False로 Convert 하지 않는 것을 볼 수 있다. 

``` go
// /go/src/github.com/influxdata/telegraf/plugins/parsers/json/parser.go 파일의 일부.
// FlattenJSON flattens nested maps/interfaces into a fields map (ignoring bools and string)
func (f *JSONFlattener) FlattenJSON(
	fieldname string,
	v interface{}) error {
	if f.Fields == nil {
		f.Fields = make(map[string]interface{})
	}
	return f.FullFlattenJSON(fieldname, v, false, false)
}

// FullFlattenJSON flattens nested maps/interfaces into a fields map (including bools and string)
func (f *JSONFlattener) FullFlattenJSON(
	fieldname string,
	v interface{},
	convertString bool,
	convertBool bool,
) error {
	if f.Fields == nil {
		f.Fields = make(map[string]interface{})
	}
	fieldname = strings.Trim(fieldname, "_")
	switch t := v.(type) {
	case map[string]interface{}:
		for k, v := range t {
			err := f.FullFlattenJSON(fieldname+"_"+k+"_", v, convertString, convertBool)
			if err != nil {
				return err
			}
		}
	case []interface{}:
		for i, v := range t {
			k := strconv.Itoa(i)
			err := f.FullFlattenJSON(fieldname+"_"+k+"_", v, convertString, convertBool)
			if err != nil {
				return nil
			}
		}
	case float64:
		f.Fields[fieldname] = t
	case string:
		if convertString {
			f.Fields[fieldname] = v.(string)
		} else {
			return nil
		}
	case bool:
		if convertBool {
			f.Fields[fieldname] = v.(bool)
		} else {
			return nil
		}
	case nil:
		return nil
	default:
		return fmt.Errorf("JSON Flattener: got unexpected type %T with value %v (%s)",
			t, t, fieldname)
	}
	return nil
}
```

/app/puser/go/src/github.com/influxdata/telegraf/plugins/inputs/amqp_consumer/amqp_consumer.go


### customizing

위의 문제를 인지 했으면 자신의 데이터 형태에 맞게 호출 부분을 고치면 된다. 이렇게 한번 코드를 수정하게 되면 go make로 telegraf 실행파일을 다시만들어야 하는데 아래의 절차를 따르면 수정된 소스코드로 컴파일된 실행파일을 얻을 수 있다.

1. go lang, **go dependency manager** 가 설치되었는지 확인한다. 
2. GOROOT, GOPATH 가 제대로 잡혀있는지 확인하고 잡혀있지않다면 아래와 같이 명령어를 입력해 준다.
	- `export GOROOT=/golang/go` (go가 설치된 위치)
	- `export GOPATH=/go` (bin, pkg, src 를 가지고 있는 곳. 즉 go 워크 스페이스)
3. 1,2 단계가 완료되었다면 소스가 존재하는 `/go/src/github.com/influxdata/telegraf` 로 이동해서 `make` 를 입력해서 수정된 코드로 동작하는 telegraf 실행파일을 생성시켜준다.
	- 해당 디렉토리에 telegraf 라는 이름의 실행파일이 있다면 오류가 날 수 있다. (이름이 같기 때문)
	- 이럴 땐 다른 폴더에 기존 telegraf 실행파일을 옮겨두거나 이름을 바꿔 두자.

위 3단계면 customize 된 telegraf 를 얻을 수 있다. 이 파일로 실행시켜 주자. 원하는대로 데이터가 적재되는 것을 확인할 수 있다.

```
./telegraf --config ./telegraf.conf
```


### 부록: Telegraf - Input Plugins

Telegraf input plugins are used with the InfluxData time series platform to collect metrics from the system, services, or third party APIs. All metrics are gathered from the inputs you enable and configure in the configuration file.

Note: Telegraf plugins added in the current release are noted with -- NEW in v1.7. The Release Notes/Changelog has a list of new plugins and updates for other plugins. See the plugin README files for more details.

Usage instructions
View usage instructions for each service input by running telegraf --usage <service-input-name>.

Supported Telegraf input plugins
Aerospike (aerospike)
The Aerospike (aerospike) input plugin queries Aerospike servers and gets node statistics and statistics for all configured namespaces.

AMQP Consumer (amqp_consumer)
The AMQP Consumer (amqp_consumer) input plugin provides a consumer for use with AMQP 0-9-1, a prominent implementation of this protocol being RabbitMQ.

Apache (apache)
The Apache (apache) input plugin collects server performance information using the mod_status module of the Apache HTTP Server.

Typically, the mod_status module is configured to expose a page at the /server-status?auto location of the Apache server. The ExtendedStatus option must be enabled in order to collect all available fields. For information about how to configure your server reference the module documenation.

Aurora (aurora) – NEW in v.1.7
The Aurora input plugin gathers metrics from Apache Aurora schedulers. For monitoring recommendations, see Monitoring your Aurora cluster.

Bcache (bcache)
The Bcache (bcache) input plugin gets bcache statistics from the stats_total directory and dirty_data file.

Bond (bond)
The Bond (bond) input plugin collects network bond interface status, bond’s slaves interfaces status and failures count of bond’s slaves interfaces. The plugin collects these metrics from /proc/net/bonding/* files.

Burrow (burrow) – NEW in v.1.7
The Burrow (burrow input plugin) collects Apache Kafka topic, consumer, and partition status using the Burrow HTTP HTTP Endpoint.

Ceph Storage (ceph)
The Ceph Storage (ceph) input plugin collects performance metrics from the MON and OSD nodes in a Ceph storage cluster.

CGroup (cgroup)
The CGroup (cgroup) input plugin captures specific statistics per cgroup.

Chrony (chrony)
The Chrony (chrony) input plugin gets standard chrony metrics, requires chronyc executable.

CloudWatch (cloudwatch)
The CloudWatch (cloudwatch) input plugin pulls metric statistics from Amazon CloudWatch.

Conntrack (conntrack)
The Conntrack (conntrack) input plugin collects stats from Netfilter’s conntrack-tools.

The conntrack-tools provide a mechanism for tracking various aspects of network connections as they are processed by netfilter. At runtime, conntrack exposes many of those connection statistics within /proc/sys/net. Depending on your kernel version, these files can be found in either /proc/sys/net/ipv4/netfilter or /proc/sys/net/netfilter and will be prefixed with either ip_ or nf_. This plugin reads the files specified in its configuration and publishes each one as a field, with the prefix normalized to ip_.

Consul (consul)
The Consul (consul) input plugin will collect statistics about all health checks registered in the Consul. It uses Consul API to query the data. It will not report the telemetry but Consul can report those stats already using StatsD protocol, if needed.

Couchbase (couchbase)
The Couchbase (couchbase) input plugin reads per-node and per-bucket metrics from Couchbase.

CouchDB (couchdb)
The CouchDB (couchdb) input plugin gathers metrics of CouchDB using _stats endpoint.

Mesosphere DC/OS (dcos)
The Mesosphere DC/OS (dcos) input plugin gathers metrics from a DC/OS cluster’s metrics component.

Disque (disque)
The Disque (disque) input plugin gathers metrics from one or more Disque servers.

DMCache (dmcache)
The DMCache (dmcache) input plugin provides a native collection for dmsetup-based statistics for dm-cache.

DNS query time (dns_query_time)
The DNS query time (dns_query_time) input plugin gathers dns query times in milliseconds - like Dig.

Docker (docker)
The Docker (docker) input plugin uses the Docker Engine API to gather metrics on running Docker containers. The Docker plugin uses the Official Docker Client to gather stats from the Engine API library documentation.

Dovecot (dovecot)
The Dovecot (dovecot) input plugin uses the dovecot Stats protocol to gather metrics on configured domains. For more information, see the Dovecot documentation.

Elasticsearch (elasticsearch)
The Elasticsearch (elasticsearch) input plugin queries endpoints to obtain node and optionally cluster-health or cluster-stats metrics.

Exec (exec)
The Exec (exec) input plugin parses supported Telegraf input data formats (InfluxDB Line Protocol, JSON, Graphite, Value, Nagios, Collectd, and Dropwizard into metrics. Each Telegraf metric includes the measurement name, tags, fields, and timesamp. See Telegraf input data formats for details on the supported data formats.

Fail2ban (fail2ban)
The Fail2ban (fail2ban) input plugin gathers the count of failed and banned ip addresses using fail2ban.

Fibaro (fibaro) – NEW in v.1.7
The Fibaro (fibaro) input plugin makes HTTP calls to the Fibaro controller API to gather values of hooked devices. Those values could be true (1) or false (0) for switches, percentage for dimmers, temperature, etc.

Filestat (filestat)
The Filestat (filestat) input plugin gathers metrics about file existence, size, and other stats.

Fluentd (fluentd)
The Fluentd (fluentd) input plugin gathers metrics from plugin endpoint provided by in_monitor plugin. This plugin understands data provided by /api/plugin.json resource (/api/config.json is not covered).

Graylog (graylog)
The Graylog (graylog) input plugin can collect data from remote Graylog service URLs. This plugin currently supports two types of endpoints:

multiple (e.g., http://[graylog-server-ip]:12900/system/metrics/multiple)
namespace (e.g., http://[graylog-server-ip]:12900/system/metrics/namespace/{namespace})
HAproxy (haproxy)
The HAproxy (haproxy) input plugin gathers metrics directly from any running HAproxy instance. It can do so by using CSV generated by HAproxy status page or from admin sockets.

Hddtemp (hddtemp)
The Hddtemp (hddtemp) input plugin reads data from hddtemp daemons.

HTTP (http)
The HTTP (http) input plugin collects metrics from one or more HTTP(S) endpoints. The endpoint should have metrics formatted in one of the supported input data formats. Each data format has its own unique set of configuration options which can be added to the input configuration.

HTTP Listener (http_listener)
The HTTP Listener (http_listener) input plugin listens for messages sent via HTTP POST. Messages are expected in the InfluxDB line protocol format ONLY (other Telegraf input data formats are not supported). The plugin allows Telegraf to serve as a proxy/router for the /write endpoint of the InfluxDB HTTP API.

HTTP Response (http_response)
The HTTP Response (http_response) input plugin gathers metrics for HTTP responses. The measurements and fields include response_time, http_response_code, and result_type. Tags for measurements include server and method.

InfluxDB (influxdb)
The InfluxDB (influxdb) gathers metrics from the exposed /debug/vars endpoint. Using Telegraf to extract these metrics to create a “monitor of monitors” is a best practice and allows you to reduce the overhead associated with capturing and storing these metrics locally within the _internal database for production deployments. Read more about this approach here.

Internal (internal)
The Internal (internal) input plugin collects metrics about the Telegraf agent itself. Note that some metrics are aggregates across all instances of one type of plugin.

Interrupts (interrupts)
The Interrupts (interrupts) input plugin gathers metrics about IRQs, including interrupts (from /proc/interrupts) and soft_interrupts (from /proc/softirqs).

IPMI Sensor (ipmi_sensor)
The IPMI Sensor (ipmi_sensor) input plugin queries the local machine or remote host sensor statistics using the impitool utility.

ipset (ipset)
The Ipset (ipset) input plugin gathers packets and bytes counters from Linux ipset. It uses the output of the command ipset save. Ipsets created without the counters option are ignored.

IPtables (iptables)
The IPtables (iptables) input plugin gathers packets and bytes counters for rules within a set of table and chain from the Linux’s iptables firewall.

Jolokia2 Agent (jolokia2_agent)
The Jolokia2 Agent (jolokia2_agent) input plugin reads JMX metrics from one or more Jolokia agent REST endpoints using the JSON-over-HTTP protocol.

Jolokia2 Proxy (jolokia2_proxy)
The Jolokia2 Proxy (jolokia2_proxy) input plugin eads JMX metrics from one or more targets by interacting with a Jolokia proxy REST endpoint using the Jolokia JSON-over-HTTP protocol.

JTI OpenConfig Telemetry – NEW in v.1.7
The JTI OpenConfig Telemetry (jti_openconfig_telemetry) input plugin reads Juniper Networks implementation of OpenConfig telemetry data from listed sensors using the Junos Telemetry Interface. Refer to openconfig.net for more details about OpenConfig and Junos Telemetry Interface (JTI).

Kafka Consumer (kafka_consumer)
The Kafka Consumer (kafka_consumer) input plugin polls a specified Kafka topic and adds messages to InfluxDB. Messages are expected in the line protocol format. Consumer Group is used to talk to the Kafka cluster so multiple instances of Telegraf can read from the same topic in parallel.

Kapacitor (kapacitor)
The Kapacitor (kapacitor) input plugin will collect metrics from the given Kapacitor instances.

Kubernetes (kubernetes)
Note: The Kubernetes input plugin is experimental and may cause high cardinality issues with moderate to large Kubernetes deployments.

The Kubernetes (kubernetes) input plugin talks to the kubelet API using the /stats/summary endpoint to gather metrics about the running pods and containers for a single host. It is assumed that this plugin is running as part of a daemonset within a Kubernetes installation. This means that Telegraf is running on every node within the cluster. Therefore, you should configure this plugin to talk to its locally running kubelet.

LeoFS (leofs) – NEW in v.1.7
The LeoFS (leofs) input plugin gathers metrics of LeoGateway, LeoManager, and LeoStorage using SNMP. See System Monitoring in the LeoFS Documentation for more information.

Lustre2 (lustre2)
Lustre Jobstats allows for RPCs to be tagged with a value, such as a job’s ID. This allows for per job statistics. The Lustre2 (lustre2) input plugin collects statistics and tags the data with the jobid.

Logparser (logparser)
The Logparser (logparser) input plugin streams and parses the given logfiles. Currently, it has the capability of parsing “grok” patterns from logfiles, which also supports regex patterns.

Mailchimp (mailchimp)
The Mailchimp (mailchimp) input plugin gathers metrics from the /3.0/reports MailChimp API.

Mcrouter (mcrouter) – NEW in v.1.7
The mcrouter (mcrouter) input plugin gathers statistics data from a mcrouter instance. Mcrouter is a memcached protocol router, developed and maintained by Facebook, for scaling memcached (http://memcached.org/) deployments. It’s a core component of cache infrastructure at Facebook and Instagram where mcrouter handles almost 5 billion requests per second at peak.

Memcached (memcached)
The Memcached (memcached) input plugin gathers statistics data from a Memcached server.

Mesos (mesos)
The Mesos (mesos) input plugin gathers metrics from Mesos. For more information, please check the Mesos Observability Metrics page.

Minecraft (minecraft)
The Minecraft (minecraft) input plugin uses the RCON protocol to collect statistics from a scoreboard on a Minecraft server.

MongoDB (mongodb)
The MongoDB (mongodb) input plugin collects mongodb stats exposed by serverStatus and few more and create a single measurement containing values.

MQTT Consumer (mqtt_consumer)
The MQTT Consumer (mqtt_consumer) input plugin reads from specified MQTT topics and adds messages to InfluxDB. Messages are in the Telegraf Input Data Formats.

MySQL (mysql)
The MySQL (mysql) input plugin gathers the statistics data from MySQL servers.0

NATS Server Monitoring (nats)
The NATS Server Monitoring (nats) input plugin gathers metrics when using the NATS Server monitoring server.

NATS Consumer (nats_consumer)
The NATS Consumer (nats_consumer) input plugin reads from specified NATS subjects and adds messages to InfluxDB. Messages are expected in the Telegraf Input Data Formats. A Queue Group is used when subscribing to subjects so multiple instances of Telegraf can read from a NATS cluster in parallel.

Network Response (net_response)
The Network Response (net_response) input plugin tests UDP and TCP connection response time. It can also check response text.

NGINX (nginx)
The NGINX (nginx) input plugin reads NGINX basic status information (ngx_http_stub_status_module).

NGINX Plus (nginx_plus)
The NGINX Plus (nginx_plus) input plugin is for NGINX Plus, the commercial version of the open source web server NGINX. To use this plugin you will need a license. For more information, see What’s the Difference between Open Source NGINX and NGINX Plus?.

Structures for NGINX Plus have been built based on history of status module documentation.

NSQ (nsq)
NSQ Consumer (nsq_consumer)
The NSQ Consumer (nsq_consumer) input plugin polls a specified NSQD topic and adds messages to InfluxDB. This plugin allows a message to be in any of the supported data_format types.

Nstat (nstat)
The Nstat (nstat) input plugin collects network metrics from /proc/net/netstat, /proc/net/snmp, and /proc/net/snmp6 files.

NTPq (ntpq)
The NTPq (ntpq) input plugin gets standard NTP query metrics, requires ntpq executable.

NVIDIA SMI (nvidia-smi) – NEW in v.1.7
The NVIDIA SMI (nvidia-smi) input plugin uses a query on the NVIDIA System Management Interface (nvidia-smi) binary to pull GPU stats including memory and GPU usage, temp and other.

OpenLDAP (openldap)
The OpenLDAP (openldap) input plugin gathers metrics from OpenLDAP’s cn=Monitor backend.

OpenSMTPD (opensmtpd)
The OpenSMTPD (opensmtpd) input plugin gathers stats from OpenSMTPD, a free implementation of the server-side SMTP protocol.

Particle.io Webhooks (particle)
The Particle.io Webhooks (particle) input plugin enables Particle webhooks to gather Particle device data which gets routed through the Particle Device Cloud into InfluxDB. For more information on the Particle webhook integration, see InfluxData Integration.

Passenger (passenger)
The Passenger (passenger) input plugin gets phusion passenger statistics using their command line utility passenger-status.

PF (pf)
The PF (pf) input plugin gathers information from the FreeBSD/OpenBSD pf firewall. Currently it can retrive information about the state table: the number of current entries in the table, and counters for the number of searches, inserts, and removals to the table. The pf plugin retrieves this information by invoking the pfstat command.

PHP FPM (phpfpm)
The PHP FPM (phpfpm) input plugin gets phpfpm statistics using either HTTP status page or fpm socket.

Ping (ping)
The Ping (ping) input plugin measures the round-trip for ping commands, response time, and other packet statistics.

Postfix (postfix)
The Postfix (postfix) input plugin reports metrics on the postfix queues. For each of the active, hold, incoming, maildrop, and deferred queues, it will report the queue length (number of items), size (bytes used by items), and age (age of oldest item in seconds).

PostgreSQL (postgresql)
The PostgreSQL (postgresql) input plugin provides metrics for your PostgreSQL database. It currently works with PostgreSQL versions 8.1+. It uses data from the built in pg_stat_database and pg_stat_bgwriter views. The metrics recorded depend on your version of postgres.

PostgreSQL Extensible (postgresql_extensible)
This PostgreSQL Extensible (postgresql_extensible) input plugin provides metrics for your postgres database. It has been designed to parse SQL queries in the plugin section of telegraf.conf files.

PowerDNS (powerdns)
The PowerDNS (powerdns) input plugin gathers metrics about PowerDNS using UNIX sockets.

Procstat (procstat)
The Procstat (procstat) input plugin can be used to monitor system resource usage by an individual process using their /proc data.

Processes can be specified either by pid file, by executable name, by command line pattern matching, by username, by systemd unit name, or by cgroup name/path (in this order or priority). This plugin uses pgrep when an executable name is provided to obtain the pid. The Procstat plugin transmits IO, memory, cpu, file descriptor-related measurements for every process specified. A prefix can be set to isolate individual process specific measurements.

The plugin will tag processes according to how they are specified in the configuration. If a pid file is used, a “pidfile” tag will be generated. On the other hand, if an executable is used an “exe” tag will be generated.

Prometheus (prometheus)
The Prometheus (prometheus) input plugin input plugin gathers metrics from HTTP servers exposing metrics in Prometheus format.

PuppetAgent (puppetagent)
The PuppetAgent (puppetagent) input plugin collects variables outputted from the last_run_summary.yaml file usually located in /var/lib/puppet/state/ PuppetAgent Runs.

RabbitMQ (rabbitmq)
The RabbitMQ (rabbitmq) input plugin reads metrics from RabbitMQ servers via the Management Plugin.

Raindrops (raindrops)
The Raindrops (raindrops) input plugin reads from specified Raindops middleware URI and adds the statistics to InfluxDB.

Redis (redis)
The Redis (redis) input plugin gathers the results of the INFO Redis command. There are two separate measurements: redis and redis_keyspace, the latter is used for gathering database-related statistics.

Additionally the plugin also calculates the hit/miss ratio (keyspace_hitrate) and the elapsed time since the last RDB save (rdb_last_save_time_elapsed).

RethinkDB (rethinkdb)
The RethinkDB (rethinkdb) input plugin works with RethinkDB 2.3.5+ databases that requires username, password authorization, and Handshake protocol v1.0.

Riak (riak)
The Riak (riak) input plugin gathers metrics from one or more Riak instances.

Salesforce (salesforce)
The Salesforce (salesforce) input plugin gathers metrics about the limits in your Salesforce organization and the remaining usage. It fetches its data from the limits endpoint of the Salesforce REST API.

Sensors (sensors)
The Sensors (sensors) input plugin collects collects sensor metrics with the sensors executable from the lm-sensor package.

SMART (smart)
The SMART (smart) input plugin gets metrics using the command line utility smartctl for S.M.A.R.T. (Self-Monitoring, Analysis and Reporting Technology) storage devices. SMART is a monitoring system included in computer hard disk drives (HDDs) and solid-state drives (SSDs), which include most modern ATA/SATA, SCSI/SAS and NVMe disks. The plugin detects and reports on various indicators of drive reliability, with the intent of enabling the anticipation of hardware failures. See smartmontools.

SNMP (snmp)
The SNMP (snmp) input plugin gathers metrics from SNMP agents.

Socket Listener (socket_listener)
The Socket Listener (socket_listener) input plugin listens for messages from streaming (tcp, unix) or datagram (UDP, unixgram) protocols. Messages are expected in the Telegraf Input Data Formats.

Solr (solr)
The Solr (solr) input plugin collects stats using the MBean Request Handler.

SQL Server (sqlserver)
The SQL Server (sqlserver) input plugin provides metrics for your Microsoft SQL Server instance. It currently works with SQL Server versions 2008+. Recorded metrics are lightweight and use Dynamic Management Views supplied by SQL Server.

StatsD (statsd)
The StatsD (statsd) input plugin is a special type of plugin which runs a backgrounded statsd listener service while Telegraf is running. StatsD messages are formatted as described in the original etsy statsd implementation.

Syslog (syslog) – NEW in v.1.7
The Syslog (syslog) input plugin listens for syslog messages transmitted over UDP or TCP. Syslog messages should be formatted according to RFC 5424.

Sysstat (sysstat)
The Sysstat (sysstat) input plugin collects sysstat system metrics with the sysstat collector utility sadc and parses the created binary data file with the sadf utility.

System (system)
The System (system) input plugin gathers general stats on system load, uptime, and number of users logged in. It is basically equivalent to the UNIX uptime command.

Tail (tail)
The Tail (tail) input plugin “tails” a logfile and parses each log message.

Teamspeak 3 (teamspeak)
The Teamspeak 3 (teamspeak) input plugin uses the Teamspeak 3 ServerQuery interface of the Teamspeak server to collect statistics of one or more virtual servers.

Tomcat (tomcat)
The Tomcat (tomcat) input plugin collects statistics available from the tomcat manager status page from the http://<host>/manager/status/all?XML=true URL. (XML=true will return only XML data). See the Tomcat documentation for details of these statistics.

Trig (trig)
The Trig (trig) input plugin inserts sine and cosine waves for demonstration purposes.

Twemproxy (twemproxy)
The Twemproxy (twemproxy) input plugin gathers data from Twemproxy instances, processes Twemproxy server statistics, processes pool data. and processes backend server (Redis/Memcached) statistics.

Unbound (unbound)
The Unbound (unbound) input plugin gathers stats from Unbound, a validating, recursive, and caching DNS resolver.

Varnish (varnish)
The Varnish (varnish) input plugin gathers stats from Varnish HTTP Cache.

Webhooks (webhooks)
The Webhooks (webhooks) input plugin starts an HTTPS server and registers multiple webhook listeners.

Win_perf_counters (win_perf_counters)
The way the Win_perf_counters (win_perf_counters) input plugin works is that on load of Telegraf, the plugin will be handed configuration from Telegraf. This configuration is parsed and then tested for validity such as if the Object, Instance and Counter existing. If it does not match at startup, it will not be fetched. Exceptions to this are in cases where you query for all instances “”. By default the plugin does not return _Total when it is querying for all () as this is redundant.

Win_services (win_services)
The Win_services (win_services) input plugin reports Windows services info.

ZFS (zfs)
The ZFS (zfs) input plugin provides metrics from your ZFS filesystems. It supports ZFS on Linux and FreeBSD. It gets ZFS statistics from /proc/spl/kstat/zfs on Linux and from sysctl and zpool on FreeBSD.

Zipkin (zipkin)
The Zipkin (zipkin) input plugin implements the Zipkin HTTP server to gather trace and timing data needed to troubleshoot latency problems in microservice architectures.

Note: This plugin is experimental. Its data schema may be subject to change based on its main usage cases and the evolution of the OpenTracing standard.

Zookeeper (zookeeper)
The Zookeeper (zookeeper) input plugin collects variables outputted from the mntr command Zookeeper Admin.
