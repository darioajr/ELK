# ELK settings to monitor Fortigate Logs
==============================================

![](https://github.com/darioajr/ELK/fortigate-logging-flow-elasticsearch.png)

## Tools
---------------------------------------
* ELK 
	* [How to Install ELK docker] (https://deviantony.wordpress.com/2014/12/06/an-elk-stack-powered-by-docker-and-fig/)
* Fortigate


### Config Fortigate SysLog
---------------------------------------
The first step was to configure the Fortigates. Here’s the Syslog format:
```
config log syslogd setting
    set status enable
    set server "192.168.xxx.xxx"
    set port 514
end
config log syslogd filter
    set severity error
end
```

One thing to note is that the log level is set to error. The reason for this is that by default, the Fortigate systems will log all sessions via syslog and this will result in a significant amount of data. Storing session data in Elasticsearch was generating hundreds of gigabytes a week and taking a considerable amount of resources to do so. For now, we're simply using the NetFlow data through other software to conduct IP accounting and detailed network analysis.

### Config Logstash
---------------------------------------
On the Logstash server, there are two elements to the configuration. The first is the overall config:

```
filter {
  if [type] == "syslog" {
    grok {
     patterns_dir => ["/etc/logstash/patterns/"]
      match => ["message" , "%{FORTIGATE_52BASE} %{FORTIGATE_52IPS}"]
      add_tag => ["fortigate"] 
    }
    grok {
       patterns_dir => ["/etc/logstash/patterns/"]
       match => ["message" , "%{FORTIGATE_52BASEV2} %{FORTIGATE_52DOS}"]
    }
  } 

output {
  elasticsearch_http {
    host => "IP of the Elasticsearch Server"
  }
}
```

Essentially, this is a fairly vanilla Logstash file. Most of the work is done in the log patterns. This is in a grok format, which takes a bit of getting used to but if you’ve worked with regular expressions before then this is very close. These patterns turn unstructured data into structured data so that it can be easily logged and therefore queried.

I found a few other references where others had used Logstash to log the Fortigate data, however none of them worked perfectly for me. Rather than stopping and starting Logstash to debug getting the patterns to work, there’s a handy Grok Debugger online, which is immensely helpful when first getting started.

With a few hours of fiddling, I managed to get a working solution:

```
FORTIDATE %{YEAR:year}\-%{MONTHNUM:month}\-%{MONTHDAY:day}

FORTIGATE_52BASE <%{NUMBER:syslog_index}>date=%{FORTIDATE:date} time=%{TIME:time} devname=%{HOST:hostname} devid=%{HOST:devid} logid=%{NUMBER:logid} type=%{WORD:type} subtype=%{WORD:subtype} eventtype=%{WORD:eventtype} level=%{WORD:level} vd=\"%{WORD:vdom}\"

FORTIGATE_52BASEV2 <%{NUMBER:syslog_index}>date=%{FORTIDATE:date} time=%{TIME:time} devname=%{HOST:hostname} devid=%{HOST:devid} logid=%{NUMBER:logid} type=%{WORD:type} subtype=%{WORD:subtype} level=%{WORD:level} vd=\"%{WORD:vdom}\"

FORTIGATE_52IPS severity=%{WORD:severity} srcip=%{IP:srcip} dstip=%{IP:dstip} sessionid=%{NUMBER:sessionid} action=%{DATA:action} proto=%{NUMBER:proto} service=%{DATA:service} attack=\"%{DATA:attack}\" srcport=%{NUMBER:srcport} dstport=%{NUMBER:dstport} attackid=%{NUMBER:attackid} profile=\"%{DATA:profile}\" ref=\"%{DATA:ref}\";? incidentserialno=%{NUMBER:incidentserialno} msg=\"%{GREEDYDATA:msg}\"

FORTIGATE_52DOS severity=%{WORD:severity} srcip=%{IP:srcip} dstip=%{IP:dstip} sessionid=%{NUMBER:sessionid} action=%{DATA:action} proto=%{NUMBER:proto} service=%{DATA:service} srcintf=\"%{HOST:srcintf}\" count=%{NUMBER:count} attack=\"%{DATA:attack}\" srcport=%{NUMBER:srcport} dstport=%{NUMBER:dstport} attackid=%{NUMBER:attackid} profile=\"%{DATA:profile}\" ref=\"%{DATA:ref}\";? msg=\"%{GREEDYDATA:msg}\" crscore=%{NUMBER:crscore} craction=%{NUMBER:craction}
```

Many of the problems came from the fact that the log format seems to change just enough to be annoying. Some items are in quotes whereas others aren’t, so this took a further few hours to sort out. Hopefully, anyone who hasn’t tried it yet can benefit from all the fiddling I’ve already had to do!

If I spend a bit more time I could probably combine a few more of the patterns into one, however once I had it working perfectly I didn’t want to touch it.


