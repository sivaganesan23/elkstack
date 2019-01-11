---------------------------------------------------------------------------------------------------------------------
# Case1
---------------------------------------------------------------------------------------------------------------------

We are going to push `/var/log/messages` to logstash.

### Beats Configuration

```
# cat filebeat.yml | grep -v '#' | grep -v '^$'
filebeat.prospectors:
- input_type: log
  paths:
    - /var/log/messages
output.logstash:
  hosts: ["elk:5044"]

# systemctl enable filebeat
# systemctl start filebeat
```

### LogStash Configuration.

```
# cat /etc/logstash/conf.d/system.conf
input {
  beats {
    port => 5044
  }
}

filter {
  grok {
    match => { "message" => "%{MONTH:month} %{MONTHDAY:mday} %{HOUR:hour}:%{MINUTE:minute}:%{SECOND:second} %{HOSTNAME:HOSTNAME} %{GREEDYDATA:logmessage}" }
  }
}

output {
  elasticsearch {
    hosts => "localhost:9200"
  }
}

```


---------------------------------------------------------------------------------------------------------------------
# Case2
---------------------------------------------------------------------------------------------------------------------

We are going to push `/var/log/messages` and `/home/studentapp/apache-tomcat-9.0.12/logs/catalina.out` to logstash.

### Beats Configuration

```
# cat filebeat.yml | grep -v '#' | grep -v '^$'
filebeat.prospectors:
- input_type: log
  paths:
    - /var/log/messages
  fields:
    logtype: system
- input_type: log
  paths:
    - /home/studentapp/apache-tomcat-9.0.12/logs/catalina.out 
  fields:
    logtype: tomcat
  
output.logstash:
  hosts: ["elk:5044"]

# systemctl enable filebeat
# systemctl start filebeat
```

### LogStash Configuration.

```
# cat /etc/logstash/conf.d/system.conf
input {
  beats {
    port => 5044
  }
}

filter {
  if [fields][logtype] == "system" {
  	grok {
    		match => { "message" => "%{MONTH:month} %{MONTHDAY:mday} %{HOUR:hour}:%{MINUTE:minute}:%{SECOND:second} %{HOSTNAME:HOSTNAME} %{GREEDYDATA:logmessage}" }
  	    }
        mutate {
		add_field => ["index_name", "%{[fields][logtype]}"]
            }
	}	
  else if [fields][logtype] == "tomcat" {
        grok {
                match => { "message" => "%{MONTHDAY:mday}-%{MONTH:month}-%{YEAR:year} %{HOUR:hour}:%{MINUTE:minute}:%{SECOND:second}\.%{INT} %{WORD:severity} %{GREEDYDATA}" }
            }   
        mutate {
                add_field => ["index_name", "%{[fields][logtype]}"]
            }
        }
}

output {
  elasticsearch {
    hosts => "localhost:9200"
    index => ["%{index_name}-%{+YYYY.MM.dd}"]
  }
}

# systemctl restart logstash
```


---------------------------------------------------------------------------------------------------------------------
# Case3
---------------------------------------------------------------------------------------------------------------------

We are going to push `/var/log/messages` and `/home/studentapp/apache-tomcat-9.0.12/logs/catalina.out` to logstash, But java exceptions is going to be multiline and hence we are goign to handle it wuth mutliline arguments in filebeat.

### Beats Configuration

```
filebeat.prospectors:
- input_type: log
  paths:
    - /var/log/messages
  fields:
    logtype: system
- input_type: log
  paths:
    - /home/studentapp/apache-tomcat-9.0.12/logs/catalina.out 
  fields:
    logtype: tomcat
  multiline.pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'
  multiline.negate: true
  multiline.match: after
  
output.logstash:
  hosts: ["elk:5044"]
```
