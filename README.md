## Elasticsearch nori 분석기 테스트 
Elasticsearch Docker 설치 및 nori(mecab) 테스트 

### Elasticsearch 7.2 / Kibana 7.2 설치 (Docker)

```bash
# elasticsearch
docker run -d --name elasticsearch  -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.2.0

# kibana
docker run -d --name kibana --link elasticsearch:elasticsearch  -e "ELASTICSEARCH_URL=http://elasticsearch:9200" -p 5601:5601 docker.elastic.co/kibana/kibana:7.2.0
```

#### # Nori 분석기 설치 
- https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-nori.html
	

```bash
# docker 내 shell 실행 
docker exec -it  elasticsearch /bin/bash

# analysis-nori 설치 
# restart 필요 
bin/elasticsearch-plugin install analysis-nori

# 사용자 사전 추가 
# 개행으로 단어를 추가하면 됩
touch config/userdict_ko.txt

# "아버"라는 사용자 단어 등록
cat "아버" >> config/userdict_ko.txt
```

### Nori Custom 분석기 세팅 
- settings : nori_tokenizer를 기반으로한 custom 분석기 등록 
- index mappings : "text" 필드에 nori_analyzer 세팅 
	- ※ 6.4 버전 이후, type의 개념이 없어짐
- https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-nori-tokenizer.html

```http
PUT nori_test
{
  "settings": {
    "index": {
      "analysis": {
        "tokenizer": {
          "nori_user_dict": {
            "type": "nori_tokenizer",
            "decompound_mode": "mixed",
            "user_dictionary": "userdict_ko.txt"
          }
        },
        "analyzer": {
          "nori_analyzer": {
            "type": "custom",
            "tokenizer": "nori_user_dict"
          }
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text"
      },
      "text": {
        "analyzer": "nori_analyzer",
        "type": "text"
      }
    }
  }
}
```

#### # 데이터 추가 
- Type 개념이 없어져 _doc 이라고 지정 

```http

POST nori_test/_doc
{
  "text" : "아버지가 방에 들어간다",
  "name" : "아버지"
}


# 결과
{
  "_index": "nori_test",
  "_type": "_doc",
  "_id": "Euy8F2wB0ezHezvb46UW",
   ...
}
```

#### # 인덱스 Term(token) 확인

```html
GET nori_test/_doc/Euy8F2wB0ezHezvb46UW/_termvectors?fields=text


# 결과
# 사용자 단어 "아버"를 추가해서 아래와 같은 분석됨 
{
  "_index": "nori_test",
  "_type": "_doc",
  "_id": "Euy8F2wB0ezHezvb46UW",
  "_version": 1,
  "found": true,
  "took": 2,
  "term_vectors": {
    "text": {
      "field_statistics": {
        "sum_doc_freq": 8,
        "doc_count": 1,
        "sum_ttf": 8
      },
      "terms": {
        "ᆫ다": {
          "term_freq": 1,
          "tokens": [
            {
              "position": 6,
              "start_offset": 8,
              "end_offset": 12
            }
          ]
        },
        "가": {
		...
        },
        "들어가": {
		...
        },
        "들어간다": {
		...
        },
        "방": {
		...
        },
        "아버": {
		...
        },
        "에": {
		...
        },
        "지": {
		...
        }
      }
    }
  }
}
```

### Analyze 테스트 
#### # nori_analyzer 분석기 : Custom 세팅 

```http
GET nori_test/_analyze
{
  "analyzer": "nori_analyzer",
  "text" : "아버지가 방에 들어간다"
}

# 결과
{
  "tokens": [
    {
      "token": "아버",
      "start_offset": 0,
      "end_offset": 2,
      "type": "word",
      "position": 0
    },
    {
      "token": "지",
      ...
    },
    {
      "token": "가",
      ...
    },
    {
      "token": "방",
      ...
    },
    {
      "token": "에",
      ...
    },
    {
      "token": "들어간다",
      ...
    },
    {
      "token": "들어가",
      ...
    },
    {
      "token": "ᆫ다",
      ...
    }
  ]
}
```

#### # Nori 분석기

```
GET nori_test/_analyze
{
  "analyzer": "nori",
  "text" : "아버지가 방에 들어간다"
}

# 결과
{
  "tokens": [
    {
      "token": "아버지",
	  ...
    },
    {
      "token": "방",
	  ...
    },
    {
      "token": "들어가",
	  ...
    }
  ]
}

```

#### # CJK 분석기

```
GET nori_test/_analyze
{
  "analyzer": "cjk",
  "text" : "아버지가 방에 들어간다"
}

# 결과
{
  "tokens": [
    {
      "token": "아버",
 	  ...
    },
    {
      "token": "버지",
 	  ...
    },
    {
      "token": "지가",
 	  ...
    },
    {
      "token": "방에",
 	  ...
    },
    {
      "token": "들어",
 	  ...
    },
    {
      "token": "어간",
 	  ...
    },
    {
      "token": "간다",
 	  ...
    }
  ]
}

```