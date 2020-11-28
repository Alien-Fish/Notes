---
layout: post
#标题配置
title:  Elasticsearch 聚合查询
#时间配置
date:   2020-04-08 10:00:00 +0800
#大类配置
categories: 数据搜索
#小类配置
tag: ElasticSearch
---

* content
{:toc}

## 说明
Elasticsearch 聚合查询是指针对数据分析提供对数据进行统计分析功能，通过聚合查询，得到一些数据的概览，方便统计整套数据的结果，如分组查询、最大最小值、平均值。  
优点：实时性高，高性能，编写简单无需在客户端实现。

## 一、指标聚合
### 1.1.max min sum avg
```bash
#先根据目的地分桶，查询每个目的地的最大、最小、平均票价，还根据目的地天气分桶
POST kibana_sample_data_flights/_search
{
  "size": 1,
  "aggs": {
    "flight_dest": {
      "terms": {
        "field": "DestCountry",
        "size": 100
      },
      "aggs": {
        "average_price": {
          "avg": {
            "field": "AvgTicketPrice"
          }
        },
        "max_price": {
          "max": {
            "field": "AvgTicketPrice"
          }
        },
        "min_price": {
          "min": {
            "field": "AvgTicketPrice"
          }
        },
        "weather": {
          "terms": {
            "field": "DestWeather"
          }
        }
      }
    }
  }
}
```
查询结果，IT机票的max_price、min_price、average_price以及天气分类汇总
```bash
{
  "took" : 13,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 10000,
      "relation" : "gte"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "flight_dest" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 10688,
      "buckets" : [
        {
          "key" : "IT",
          "doc_count" : 2371,
          "max_price" : {
            "value" : 1195.3363037109375
          },
          "min_price" : {
            "value" : 100.57646942138672
          },
          "weather" : {
            "doc_count_error_upper_bound" : 0,
            "sum_other_doc_count" : 0,
            "buckets" : [
              {
                "key" : "Clear",
                "doc_count" : 428
              },
              {
                "key" : "Sunny",
                "doc_count" : 424
              },
              {
                "key" : "Rain",
                "doc_count" : 417
              },
              {
                "key" : "Cloudy",
                "doc_count" : 414
              },
              {
                "key" : "Heavy Fog",
                "doc_count" : 182
              },
              {
                "key" : "Damaging Wind",
                "doc_count" : 173
              },
              {
                "key" : "Hail",
                "doc_count" : 169
              },
              {
                "key" : "Thunder & Lightning",
                "doc_count" : 164
              }
            ]
          },
          "average_price" : {
            "value" : 586.9627099618385
          }
        }
      ]
    }
  }
}
```

### 1.2.count 
文档计数
```bash
POST kibana_sample_data_flights/_doc/_count
{
  "query": {
    "match": {
      "DestCountry": "CH"
    }
  }
}
```
查询结果，count查询类型已过期
```bash
#! Deprecation: [types removal] Specifying types in count requests is deprecated.
{
  "count" : 691,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  }
}

```

### 1.3.Value count 
统计某字段有值的文档数
```base
POST /kibana_sample_data_flights/_search?size=0
{
  "aggs": {
    "DestCountry_count": {
      "value_count": {
        "field": "DestCountry"
      }
    }
  }
}
```
查询结果
```base
{
  "took" : 6,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 10000,
      "relation" : "gte"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "DestCountry_count" : {
      "value" : 13059
    }
  }
}
```

### 1.4.cardinality  
值去重计数
去重统计查询国家数和天气类型数
```base
POST /kibana_sample_data_flights/_search?size=0
{
  "aggs": {
    "DestCountry_count": {
      "cardinality": {
        "field": "DestCountry"
      }
    },
    "DestWeather_count": {
      "cardinality": {
        "field": "DestWeather"
      }
    }
  }
}
```
查询结果，32个国家，8种天气
```base
{
  "took" : 4,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 10000,
      "relation" : "gte"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "DestWeather_count" : {
      "value" : 8
    },
    "DestCountry_count" : {
      "value" : 32
    }
  }
}
```

### 1.5.stats 统计
统计 count max min avg sum 5个值
```base
POST /kibana_sample_data_flights/_search?size=0
{
  "aggs": {
    "AvgTicketPrice_stats": {
      "stats": {
        "field": "AvgTicketPrice"
      }
    }
  }
}
```
查询结果，机票价格最大最小平均值
```base
{
  "took" : 5,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 10000,
      "relation" : "gte"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "AvgTicketPrice_stats" : {
      "count" : 13059,
      "min" : 100.0205307006836,
      "max" : 1199.72900390625,
      "avg" : 628.2536888148849,
      "sum" : 8204364.922233582
    }
  }
}
```

### 1.6.Extended stats 
高级统计
高级统计，比stats多4个统计结果： 平方和、方差、标准差、平均值加/减两个标准差的区间
```base
POST /kibana_sample_data_flights/_search?size=0
{
  "aggs": {
    "AvgTicketPrice_stats": {
      "extended_stats": {
        "field": "AvgTicketPrice"
      }
    }
  }
}
```
查询结果
```base
{
  "took" : 6,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 10000,
      "relation" : "gte"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "AvgTicketPrice_stats" : {
      "count" : 13059,
      "min" : 100.0205307006836,
      "max" : 1199.72900390625,
      "avg" : 628.2536888148849,
      "sum" : 8204364.922233582,
      "sum_of_squares" : 6.081113366492191E9,
      "variance" : 70961.85310632498,
      "std_deviation" : 266.38666090163935,
      "std_deviation_bounds" : {
        "upper" : 1161.0270106181636,
        "lower" : 95.48036701160618
      }
    }
  }
}
```

### 1.7.Percentiles 
占比百分位对应的值统计
```base
POST /kibana_sample_data_flights/_search?size=0
{
  "aggs": {
    "AvgTicketPrice_percentiles": {
      "percentiles": {
        "field": "AvgTicketPrice"
      }
    }
  }
}
```
查询结果，占比为50%的文档的AvgTicketPrice值 <= 640
```base
{
  "took" : 66,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 10000,
      "relation" : "gte"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "AvgTicketPrice_percentiles" : {
      "values" : {
        "1.0" : 118.31498379707335,
        "5.0" : 187.97447717898737,
        "25.0" : 409.97985471904417,
        "50.0" : 640.4095028794482,
        "75.0" : 842.2513977007038,
        "95.0" : 1037.3295444664586,
        "99.0" : 1166.955294494629
      }
    }
  }
}
```

### 1.8.Percentiles rank 
统计值小于等于指定值的文档占比
```bash
POST /kibana_sample_data_flights/_search?size=0
{
  "aggs": {
    "AvgTicketPrice_percentile_ranks": {
      "percentile_ranks": {
        "field": "AvgTicketPrice",
        "values": [
          500,
          1000
        ]
      }
    }
  }
}
```
查询结果，文档的AvgTicketPrice值 <= 500占比33%， <= 100占比93%
```base
{
  "took" : 20,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 10000,
      "relation" : "gte"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "AvgTicketPrice_percentile_ranks" : {
      "values" : {
        "500.0" : 33.81066111996899,
        "1000.0" : 93.7778918791443
      }
    }
  }
}
```

## 二、桶聚合
### 2.1.Terms 根据字段值项分组聚合 
```bash
#根据目的地分桶
POST kibana_sample_data_flights/_search
{
  "size": 1,
  "aggs": {
    "flight_dest": {
      "terms": {
        "field": "DestCountry",
        "size": 100,
        "order" : { "_count" : "asc" }
      }
    }
  }
}
```

### 2.2.Filter 对满足过滤查询的文档进行聚合计算
```bash
POST kibana_sample_data_flights/_search
{
  "size": 0,
  "aggs": {
    "age_price": {
      "filter": {"match":{"DestCountry": "CH"}},
      "aggs": {
        "average_price": {
          "avg": {
            "field": "AvgTicketPrice"
          }
        },
        "max_price": {
          "max": {
            "field": "AvgTicketPrice"
          }
        },
        "min_price": {
          "min": {
            "field": "AvgTicketPrice"
          }
        },
        "weather": {
          "terms": {
            "field": "DestWeather"
          }
        }
      }
    }
  }
}
```
查询结果，查询国家为CH的航班机票最大、最小、平均票价
```bash
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 10000,
      "relation" : "gte"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "age_price" : {
      "doc_count" : 691,
      "max_price" : {
        "value" : 1196.496826171875
      },
      "min_price" : {
        "value" : 101.3473129272461
      },
      "weather" : {
        "doc_count_error_upper_bound" : 0,
        "sum_other_doc_count" : 0,
        "buckets" : [
          {
            "key" : "Cloudy",
            "doc_count" : 135
          },
          {
            "key" : "Sunny",
            "doc_count" : 134
          },
          {
            "key" : "Clear",
            "doc_count" : 128
          },
          {
            "key" : "Rain",
            "doc_count" : 115
          },
          {
            "key" : "Heavy Fog",
            "doc_count" : 51
          },
          {
            "key" : "Hail",
            "doc_count" : 46
          },
          {
            "key" : "Damaging Wind",
            "doc_count" : 41
          },
          {
            "key" : "Thunder & Lightning",
            "doc_count" : 41
          }
        ]
      },
      "average_price" : {
        "value" : 575.1067587028537
      }
    }
  }
}
```

### 2.3.Filters 多个过滤组聚合计算
```bash
POST kibana_sample_data_flights/_search
{
  "size": 0,
  "aggs": {
    "age_price": {
      "filters": {
        "filters": {
          "weather_Clear": {
            "match": {
              "DestWeather": "Clear"
            }
          },
          "weather_Cloudy": {
            "match": {
              "DestWeather": "Cloudy"
            }
          }
        }
      },
      "aggs": {
        "average_price": {
          "avg": {
            "field": "AvgTicketPrice"
          }
        },
        "max_price": {
          "max": {
            "field": "AvgTicketPrice"
          }
        },
        "min_price": {
          "min": {
            "field": "AvgTicketPrice"
          }
        }
      }
    }
  }
}
```
查询结果，查询天气为Clear和Cloudy的航班机票最大、最小、平均票价
```bash
{
  "took" : 7,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 10000,
      "relation" : "gte"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "age_price" : {
      "buckets" : {
        "weather_Clear" : {
          "doc_count" : 2335,
          "max_price" : {
            "value" : 1199.5123291015625
          },
          "min_price" : {
            "value" : 100.37113189697266
          },
          "average_price" : {
            "value" : 626.1267689870308
          }
        },
        "weather_Cloudy" : {
          "doc_count" : 2221,
          "max_price" : {
            "value" : 1199.72900390625
          },
          "min_price" : {
            "value" : 100.14596557617188
          },
          "average_price" : {
            "value" : 633.0500552620989
          }
        }
      }
    }
  }
}
```

### 2.4.Range 范围分组聚合
```bash
POST kibana_sample_data_flights/_search
{
  "size": 0,
  "aggs": {
    "Price_range": {
      "range": {
        "field": "AvgTicketPrice",
        "ranges": [
          {
            "to": 500
          },
          {
            "from": 500,
            "to": 1000
          },
          {
            "from": 1000
          }
        ]
      },
      "aggs": {
        "average_price": {
          "avg": {
            "field": "AvgTicketPrice"
          }
        },
        "max_price": {
          "max": {
            "field": "AvgTicketPrice"
          }
        },
        "min_price": {
          "min": {
            "field": "AvgTicketPrice"
          }
        }
      }
    }
  }
}
```
查询结果，根据机票价格分段区间查询汇总最大、最小、平均票价
```bash
{
  "took" : 6,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 10000,
      "relation" : "gte"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "Price_range" : {
      "buckets" : [
        {
          "key" : "*-500.0",
          "to" : 500.0,
          "doc_count" : 4411,
          "max_price" : {
            "value" : 499.92926025390625
          },
          "min_price" : {
            "value" : 100.0205307006836
          },
          "average_price" : {
            "value" : 322.56621517958865
          }
        },
        {
          "key" : "500.0-1000.0",
          "from" : 500.0,
          "to" : 1000.0,
          "doc_count" : 7844,
          "max_price" : {
            "value" : 999.9935302734375
          },
          "min_price" : {
            "value" : 500.05316162109375
          },
          "average_price" : {
            "value" : 751.908526154576
          }
        },
        {
          "key" : "1000.0-*",
          "from" : 1000.0,
          "doc_count" : 804,
          "max_price" : {
            "value" : 1199.72900390625
          },
          "min_price" : {
            "value" : 1000.0712280273438
          },
          "average_price" : {
            "value" : 1098.9488406964203
          }
        }
      ]
    }
  }
}
```

### 2.5.Date Range 时间范围分组聚合
```bash
POST kibana_sample_data_flights/_search
{
  "size": 10,
  "aggs": {
    "Price_range": {
      "date_range": {
        "field": "timestamp",
        "format": "MM-yyy",
        "ranges": [
          {
            "to": "now-10M/M"
          },
          {
            "from": "now-10M/M"
          }
        ]
      },
      "aggs": {
        "average_price": {
          "avg": {
            "field": "AvgTicketPrice"
          }
        },
        "max_price": {
          "max": {
            "field": "AvgTicketPrice"
          }
        },
        "min_price": {
          "min": {
            "field": "AvgTicketPrice"
          }
        }
      }
    }
  }
}
```

```bash
{
  "took" : 3,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 10000,
      "relation" : "gte"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "Price_range" : {
      "buckets" : [
        {
          "key" : "*-06-2019",
          "to" : 1.5593472E12,
          "to_as_string" : "06-2019",
          "doc_count" : 0,
          "max_price" : {
            "value" : null
          },
          "min_price" : {
            "value" : null
          },
          "average_price" : {
            "value" : null
          }
        },
        {
          "key" : "06-2019-*",
          "from" : 1.5593472E12,
          "from_as_string" : "06-2019",
          "doc_count" : 13059,
          "max_price" : {
            "value" : 1199.72900390625
          },
          "min_price" : {
            "value" : 100.0205307006836
          },
          "average_price" : {
            "value" : 628.2536888148849
          }
        }
      ]
    }
  }
}
```

### 2.6.Date Histogram 时间直方图（柱状）聚合
按天、月、年等进行聚合统计。可按 year (1y), quarter (1q), month (1M), week (1w), day (1d), hour (1h), minute (1m), second (1s) 间隔聚合或指定的时间间隔聚合。
```bash
POST kibana_sample_data_flights/_search
{
  "size": 0,
  "aggs": {
    "Price_histogram": {
      "date_histogram": {
        "field": "timestamp",
        "interval": "month"
      },
      "aggs": {
        "average_price": {
          "avg": {
            "field": "AvgTicketPrice"
          }
        },
        "max_price": {
          "max": {
            "field": "AvgTicketPrice"
          }
        },
        "min_price": {
          "min": {
            "field": "AvgTicketPrice"
          }
        }
      }
    }
  }
}
```
查询结果
```bash
#! Deprecation: [interval] on [date_histogram] is deprecated, use [fixed_interval] or [calendar_interval] in the future.
{
  "took" : 30,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 10000,
      "relation" : "gte"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "Price_histogram" : {
      "buckets" : [
        {
          "key_as_string" : "2020-03-01T00:00:00.000Z",
          "key" : 1583020800000,
          "doc_count" : 637,
          "max_price" : {
            "value" : 999.1396484375
          },
          "min_price" : {
            "value" : 101.83601379394531
          },
          "average_price" : {
            "value" : 575.1916931847014
          }
        },
        {
          "key_as_string" : "2020-04-01T00:00:00.000Z",
          "key" : 1585699200000,
          "doc_count" : 9408,
          "max_price" : {
            "value" : 1199.72900390625
          },
          "min_price" : {
            "value" : 100.0205307006836
          },
          "average_price" : {
            "value" : 623.8758731715533
          }
        },
        {
          "key_as_string" : "2020-05-01T00:00:00.000Z",
          "key" : 1588291200000,
          "doc_count" : 3014,
          "max_price" : {
            "value" : 1199.109130859375
          },
          "min_price" : {
            "value" : 101.36595153808594
          },
          "average_price" : {
            "value" : 653.1332444847224
          }
        }
      ]
    }
  }
}
```

### 2.7.Missing  缺失值的桶聚合
```bash
POST kibana_sample_data_flights/_search
{
  "size": 0,
  "aggs": {
    "Price_missing_field": {
      "missing": {
        "field": "timestamp"
      },
      "aggs": {
        "average_price": {
          "avg": {
            "field": "AvgTicketPrice"
          }
        },
        "max_price": {
          "max": {
            "field": "AvgTicketPrice"
          }
        },
        "min_price": {
          "min": {
            "field": "AvgTicketPrice"
          }
        }
      }
    }
  }
}
```
查询不包含timestamp字段文档
```bash
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 10000,
      "relation" : "gte"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "Price_missing_field" : {
      "doc_count" : 0,
      "max_price" : {
        "value" : null
      },
      "min_price" : {
        "value" : null
      },
      "average_price" : {
        "value" : null
      }
    }
  }
}
```

### 2.8.Geo Distance 地理距离分区聚合
略
