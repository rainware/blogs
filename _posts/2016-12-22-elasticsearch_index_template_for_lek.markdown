---
layout: post
title:  "logstash-template.json"
date:   2017-03-01 11:30:00 +0800
category: 配置文件
tagline: "Supporting tagline"
tags: [lek, elk, logstash, template, elasticsearch, dynamic mapping]
---

`put /_template/<template_name>``

```python
{
  "template": "meme-lek-*",
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 2
  },
  "order": 0, # 能匹配到的多个模板，会将json组合，字段相同时数值大的覆盖数值小的
  "mappings": {
    "_default_": { # 如果没有指定type的配置，则_default生效
      "dynamic_templates": [
        {
          "message_field": { # 名字怎么起都行
            "match": "message", # 精确匹配名为message的字段
            "mapping": {
              "type": "string",
              "index": "analyzed" # es将会在index的时候对message进行分词等处理
            }
          }
        },
        {
          "string_field": {
            "match": "*",
            "mapping": {
              "type": "string",
              "index": "not_analyzed"
            },
            "match_mapping_type": "string" # 所有的字符串类型
          }
        }
      ],
      "properties": {
        "@timestamp": {
          "type": "date",
          "index": "not_analyzed"
        },
        "@version": {
          "type": "string",
          "index": "not_analyzed"
        },
        "geoip": {
          "dynamic": true,
          "properties": {
            "ip": {
              "type": "ip",
              "index": "not_analyzed"
            },
            "location": {
              "type": "geo_point",
              "index": "not_analyzed"
            },
            "latitude": {
              "type": "float",
              "index": "not_analyzed"
            },
            "longitude": {
              "type": "float",
              "index": "not_analyzed"
            }
          }
        }
      }
    }
  }
}
```
