---
layout: post
title:  "[Elastalert]Elasticsearch监控告警插件"
date:   2017-03-02 12:06:00 +0800
category: 安装文档
tagline: "Supporting tagline"
tags: [lek, elk, elastalert, elasticsearch, 监控告警]
---

ElastAlert是Yelp公司开源的一套用Python2.6写的框架，用于对Elasticsearch中数据进行监控告警。官网地址见：http://elastalert.readthedocs.org/。

# 1. 安装

下载
```
$ git clone https://github.com/Yelp/elastalert.git
```

Centos系统需要:
  1. 确保安装了pip：

      ```
      $ yum -y install epel-release
      $ yum -y install python-pip
      ```
  2. 确保setuptools是最新的

    ```
    $pip install -U setuptools
    ```
  3. 确保安装了gcc和python-devel

    ```
    $ yum install -y gcc
    $ yum install -y python-devel
    ```

安装

```
$ python setup.py install
$ pip install -r requirements.txt
```

需要安装python的elasticsearch库

```
$ pip install elasticsearch
```

# 2. 配置elastalert

### 2.1 在es中创建writeback_index

elastalert在运行时会将状态数据存储在es中, 运行`elastalert-create-index`来创建索引

### 2.2 elastalert配置

创建规则目录

`$ mkdir rules`

拷贝配置文件
```
$ cp config.yaml.example config.yaml
$ vim config.yaml
```

```yaml
# This is the folder that contains the rule yaml files
# Any .yaml file will be loaded as a rule
rules_folder: rules

# How often ElastAlert will query Elasticsearch
# The unit can be anything from weeks to seconds
run_every:
  minutes: 5

# ElastAlert will buffer results from the most recent
# period of time, in case some log sources are not in real time
buffer_time:
  minutes: 15

# The Elasticsearch hostname for metadata writeback
# Note that every rule can have its own Elasticsearch host
es_host: xx.xx.xx.xx

# The Elasticsearch port
es_port: 9200

# Optional URL prefix for Elasticsearch
#es_url_prefix: elasticsearch

# Connect with TLS to Elasticsearch
#use_ssl: True

# Verify TLS certificates
#verify_certs: True

# GET request with body is the default option for Elasticsearch.
# If it fails for some reason, you can pass 'GET', 'POST' or 'source'.
# See http://elasticsearch-py.readthedocs.io/en/master/connection.html?highlight=send_get_body_as#transport
# for details
#es_send_get_body_as: GET

# Option basic-auth username and password for Elasticsearch
#es_username: someusername
#es_password: somepassword

# The index on es_host which is used for metadata storage
# This can be a unmapped index, but it is recommended that you run
# elastalert-create-index to set a mapping
writeback_index: elastalert

# If an alert fails for some reason, ElastAlert will retry
# sending the alert until this time period has elapsed
alert_time_limit:
  days: 2
```

### 2.3 配置rule

这里以flatline类型举例

```
$ vim rules/push_monitor.yaml
```

```yaml
# (Optional)
# Elasticsearch host
es_host: xx.xx.xx.xx

# (Optional)
# Elasticsearch port
es_port: 9200

# (OptionaL) Connect with SSL to Elasticsearch
#use_ssl: True

# (Optional) basic-auth username and password for Elasticsearch
#es_username: someusername
#es_password: somepassword

# (Required)
# Rule name, must be unique
name: push-monitor

# (Required)
# Type of alert.
# the frequency rule type alerts when num_events events occur with timeframe time
type: flatline

# (Required)
# Index to search, wildcard supported
index: xx-xx-*

realert:
    minutes: 10

exponential_realert:
    minutes: 60

use_count_query: true

doc_type: tomcat

threshold: 100000

timeframe:
   hours : 3

# (Required)
# A list of Elasticsearch filters used for find events
# These filters are joined with AND and nested in a filtered query
# For more info: http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl.html
filter:
- term:
    tags: "doPush"

# (Required)
# The alert is use when a match is found
alert:
- "email"

# (required, email specific)
# a list of email addresses to send alerts to
email:
- "x@xx.com"

smtp_host: x.com
smtp_port: x
smtp_auth_file: /etc/elastalert/smtp_auth.yaml
```

```
$ vim /etc/elastalert/smtp_auth.yaml
```

```yaml
user: "xxx"
password: "xxx"
```

### 2.4 测试配置

```
$ elastalert-test-rule rules/push-monitor.yaml
```

# 3. 运行

```
python -m elastalert.elastalert --rule rules/push-monitor.yaml
```
