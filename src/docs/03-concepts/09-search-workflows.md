---
layout: default
title: Search and filtering workflows
permalink: /docs/concepts/search-workflows
---

# Searching/Filtering Workflows

## Introduction

Cadence supports creating :workflow:workflows: with customized key-value pairs, updating the information within the :workflow: code, and then listing/searching :workflow:workflows: with a SQL-like :query:. For example, you can create :workflow:workflows: with keys `city` and `age`, then search all :workflow:workflows: with `city = seattle and age > 22`.

Also note that normal :workflow: properties like start time and :workflow: type can be queried as well. For example, the following :query: could be specified when [listing workflows from the CLI](/docs/06-cli/#list-closed-or-open-workflow-executions) or using the list APIs ([Go](https://godoc.org/go.uber.org/cadence/client#Client), [Java](https://static.javadoc.io/com.uber.cadence/cadence-client/2.6.0/com/uber/cadence/WorkflowService.Iface.html#ListWorkflowExecutions-com.uber.cadence.ListWorkflowExecutionsRequest-)):

```sql
WorkflowType = "main.Workflow" and CloseStatus != 0 and (StartTime > "2019-06-07T16:46:34-08:00" or CloseTime > "2019-06-07T16:46:34-08:00" order by StartTime desc)
```

## Memo vs Search Attributes

Cadence offers two methods for creating :workflow:workflows: with key-value pairs: memo and search attributes. Memo can only be provided on :workflow: start. Also, memo data are not indexed, and are therefore not searchable. Memo data are visible when listing :workflow:workflows: using the list APIs. Search attributes data are indexed so you can search :workflow:workflows: by :query:querying: on these attributes. However, search attributes require the use of Elasticsearch.

Memo and search attributes are available in the Go client in [StartWorkflowOptions](https://godoc.org/go.uber.org/cadence/internal#StartWorkflowOptions).

```go
type StartWorkflowOptions struct {
    // ...

    // Memo - Optional non-indexed info that will be shown in list workflow.
    Memo map[string]interface{}

    // SearchAttributes - Optional indexed info that can be used in query of List/Scan/Count workflow APIs (only
    // supported when Cadence server is using Elasticsearch). The key and value type must be registered on Cadence server side.
    // Use GetSearchAttributes API to get valid key and corresponding value type.
    SearchAttributes map[string]interface{}
}
```

In the Java client, the *WorkflowOptions.Builder* has similar methods for [memo](https://static.javadoc.io/com.uber.cadence/cadence-client/2.6.0/com/uber/cadence/client/WorkflowOptions.Builder.html#setMemo-java.util.Map-) and [search attributes](https://static.javadoc.io/com.uber.cadence/cadence-client/2.6.0/com/uber/cadence/client/WorkflowOptions.Builder.html#setSearchAttributes-java.util.Map-).

Some important distinctions between memo and search attributes:

- Memo can support all data types because it is not indexed. Search attributes only support basic data types (including String, Int, Float, Bool, Datetime) because it is indexed by Elasticsearch.
- Memo does not restrict on key names. Search attributes require that keys are allowlisted before using them because Elasticsearch has a limit on indexed keys.
- Memo doesn't require Cadence clusters to depend on Elasticsearch while search attributes only works with Elasticsearch.

## Search Attributes (Go Client Usage)

When using the Cadence Go client, provide key-value pairs as SearchAttributes in [StartWorkflowOptions](https://godoc.org/go.uber.org/cadence/internal#StartWorkflowOptions).

SearchAttributes is `map[string]interface{}` where the keys need to be allowlisted so that Cadence knows the attribute key name and value type. The value provided in the map must be the same type as registered.

### Allow Listing Search Attributes

Start by :query:querying: the list of search attributes using the :CLI::

```bash
$ cadence --domain samples-domain cl get-search-attr
+---------------------+------------+
|         KEY         | VALUE TYPE |
+---------------------+------------+
| CloseStatus         | INT        |
| CloseTime           | INT        |
| CustomBoolField     | DOUBLE     |
| CustomDatetimeField | DATETIME   |
| CustomDomain        | KEYWORD    |
| CustomDoubleField   | BOOL       |
| CustomIntField      | INT        |
| CustomKeywordField  | KEYWORD    |
| CustomStringField   | STRING     |
| DomainID            | KEYWORD    |
| ExecutionTime       | INT        |
| HistoryLength       | INT        |
| RunID               | KEYWORD    |
| StartTime           | INT        |
| WorkflowID          | KEYWORD    |
| WorkflowType        | KEYWORD    |
+---------------------+------------+
```

Use the admin :CLI: to add a new search attribute:

```bash
cadence --domain samples-domain adm cl asa --search_attr_key NewKey --search_attr_type 1
```

The numbers for the attribute types map as follows:

- 0 = String
- 1 = Keyword
- 2 = Int
- 3 = Double
- 4 = Bool
- 5 = DateTime

#### Keyword vs String

Note that **Keyword** and **String** are concepts taken from Elasticsearch. Each word in a **String** is considered a searchable keyword. For a UUID, that can be problematic as Elasticsearch will index each portion of the UUID separately. To have the whole string considered as a searchable keyword, use the **Keyword** type.

For example, key RunID with value "2dd29ab7-2dd8-4668-83e0-89cae261cfb1"

- as a **Keyword** will only be matched by RunID = "2dd29ab7-2dd8-4668-83e0-89cae261cfb1" (or in the future with [regular expressions](https://github.com/uber/cadence/issues/1137))
- as a **String** will be matched by RunID =  "2dd8", which may cause unwanted matches

**Note:** String type can not be used in Order By :query:.

There are some pre-allowlisted search attributes that are handy for testing:

- CustomKeywordField
- CustomIntField
- CustomDoubleField
- CustomBoolField
- CustomDatetimeField
- CustomStringField

Their types are indicated in their names.

### Value Types

Here are the Search Attribute value types and their correspondent Golang types:

- Keyword = string
- Int = int64
- Double = float64
- Bool = bool
- Datetime = time.Time
- String = string

### Limit

We recommend limiting the number of Elasticsearch indexes by enforcing limits on the following:

- Number of keys: 100 per :workflow:
- Size of value: 2kb per value
- Total size of key and values: 40kb per :workflow:

Cadence reserves keys like DomainID, WorkflowID, and RunID. These can only be used in list :query:queries:. The values are not updatable.

### Upsert Search Attributes in Workflow

[UpsertSearchAttributes](https://godoc.org/go.uber.org/cadence/workflow#UpsertSearchAttributes) is used to add or update search attributes from within the :workflow: code.

Go samples for search attributes can be found at [github.com/uber-common/cadence-samples](https://github.com/uber-common/cadence-samples/tree/master/cmd/samples/recipes/searchattributes).

UpsertSearchAttributes will merge attributes to the existing map in the :workflow:. Consider this example :workflow: code:

```go
func MyWorkflow(ctx workflow.Context, input string) error {

    attr1 := map[string]interface{}{
        "CustomIntField": 1,
        "CustomBoolField": true,
    }
    workflow.UpsertSearchAttributes(ctx, attr1)

    attr2 := map[string]interface{}{
        "CustomIntField": 2,
        "CustomKeywordField": "seattle",
    }
    workflow.UpsertSearchAttributes(ctx, attr2)
}
```

After the second call to UpsertSearchAttributes, the map will contain:

```go
map[string]interface{}{
    "CustomIntField": 2,
    "CustomBoolField": true,
    "CustomKeywordField": "seattle",
}
```

There is no support for removing a field. To achieve a similar effect, set the field to a sentinel value. For example, to remove “CustomKeywordField”, update it to “impossibleVal”. Then searching `CustomKeywordField != ‘impossibleVal’`  will match :workflow:workflows: with CustomKeywordField not equal to "impossibleVal", which **includes** :workflow:workflows: without the CustomKeywordField set.

Use `workflow.GetInfo` to get current search attributes.

### ContinueAsNew and Cron

When performing a [ContinueAsNew](/docs/go-client/continue-as-new/) or using [Cron](/docs/go-client/distributed-cron/), search attributes (and memo) will be carried over to the new run by default.

## Query Capabilities

:query:Query: :workflow:workflows: by using a SQL-like where clause when [listing workflows from the CLI](/docs/06-cli/#list-closed-or-open-workflow-executions) or using the list APIs ([Go](https://godoc.org/go.uber.org/cadence/client#Client), [Java](https://static.javadoc.io/com.uber.cadence/cadence-client/2.6.0/com/uber/cadence/WorkflowService.Iface.html#ListWorkflowExecutions-com.uber.cadence.ListWorkflowExecutionsRequest-)).

Note that you will only see :workflow:workflows: from one domain when :query:querying:.

### Supported Operators

- AND, OR, ()
- =, !=, >, >=, <, <=
- IN
- BETWEEN ... AND
- ORDER BY

### Default Attributes

More and more default attributes are added in newer versions.
Please get the  by using the :CLI: get-search-attr command or the GetSearchAttributes API. 
Some names and types are as follows:

| KEY                 | VALUE TYPE |
| ------------------- | ---------- |
| CloseStatus         | INT        |
| CloseTime           | INT        |
| CustomBoolField     | DOUBLE     |
| CustomDatetimeField | DATETIME   |
| CustomDomain        | KEYWORD    |
| CustomDoubleField   | BOOL       |
| CustomIntField      | INT        |
| CustomKeywordField  | KEYWORD    |
| CustomStringField   | STRING     |
| DomainID            | KEYWORD    |
| ExecutionTime       | INT        |
| HistoryLength       | INT        |
| RunID               | KEYWORD    |
| StartTime           | INT        |
| WorkflowID          | KEYWORD    |
| WorkflowType        | KEYWORD    |
| Tasklist            | KEYWORD    |


There are some special considerations for these attributes:

- CloseStatus, CloseTime, DomainID, ExecutionTime, HistoryLength, RunID, StartTime, WorkflowID, WorkflowType are reserved by Cadence and are read-only
- CloseStatus is a mapping of int to state:
  - 0 = completed
  - 1 = failed
  - 2 = canceled
  - 3 = terminated
  - 4 = continuedasnew
  - 5 = timedout
- StartTime, CloseTime and ExecutionTime are stored as INT, but support :query:queries: using both EpochTime in nanoseconds, and string in RFC3339 format (ex. `"2006-01-02T15:04:05+07:00"`)
- CloseTime, CloseStatus, HistoryLength are only present in closed :workflow:
- ExecutionTime is for Retry/Cron user to :query: a :workflow: that will run in the future

To list only open :workflow:workflows:, add `CloseTime = missing` to the end of the :query:.

If you use retry or the cron feature to :query: :workflow:workflows: that will start execution in a certain time range, you can add predicates on ExecutionTime. For example: `ExecutionTime > 2019-01-01T10:00:00-07:00`. Note that if predicates on ExecutionTime are included, only cron or a :workflow: that needs to retry will be returned.

### General Notes About Queries

- Pagesize default is 1000, and cannot be larger than 10k
- Range :query: on Cadence timestamp (StartTime, CloseTime, ExecutionTime) cannot be larger than 9223372036854775807 (maxInt64 - 1001)
- :query:Query: by time range will have 1ms resolution
- :query:Query: column names are case sensitive
- ListWorkflow may take longer when retrieving a large number of :workflow:workflows: (10M+)
- To retrieve a large number of :workflow:workflows: without caring about order, use the ScanWorkflow API
- To efficiently count the number of :workflow:workflows:, use the CountWorkflow API

## Tools Support

### CLI

Support for search attributes is available as of version 0.6.0 of the Cadence server. You can also use the :CLI: from the latest [CLI Docker image](https://hub.docker.com/r/ubercadence/cli) (supported on 0.6.4 or later).

#### Start Workflow with Search Attributes

```bash
cadence --do samples-domain workflow start --tl helloWorldGroup --wt main.Workflow --et 60 --dt 10 -i '"vancexu"' -search_attr_key 'CustomIntField | CustomKeywordField | CustomStringField |  CustomBoolField | CustomDatetimeField' -search_attr_value '5 | keyword1 | vancexu test | true | 2019-06-07T16:16:36-08:00'
```

#### Search Workflows with List API

```bash
cadence --do samples-domain wf list -q '(CustomKeywordField = "keyword1" and CustomIntField >= 5) or CustomKeywordField = "keyword2"' -psa
```

```bash
cadence --do samples-domain wf list -q 'CustomKeywordField in ("keyword2", "keyword1") and CustomIntField >= 5 and CloseTime between "2018-06-07T16:16:36-08:00" and "2019-06-07T16:46:34-08:00" order by CustomDatetimeField desc' -psa
```

To list only open :workflow:workflows:, add `CloseTime = missing` to the end of the :query:.

Note that :query:queries: can support more than one type of filter:

```bash
cadence --do samples-domain wf list -q 'WorkflowType = "main.Workflow" and (WorkflowID = "1645a588-4772-4dab-b276-5f9db108b3a8" or RunID = "be66519b-5f09-40cd-b2e8-20e4106244dc")'
```

```bash
cadence --do samples-domain wf list -q 'WorkflowType = "main.Workflow" StartTime > "2019-06-07T16:46:34-08:00" and CloseTime = missing'
```

### Web UI Support

:query:Queries: are supported in [Cadence Web](https://github.com/uber/cadence-web) as of release 3.4.0. Use the "Basic/Advanced" button to switch to "Advanced" mode and type the :query: in the search box.

### TLS Support for connecting to Elasticsearch

If your elasticsearch deployment requires TLS to connect to it, you can add the following to your config template.
The TLS config is optional and when not provided it defaults to tls.enabled to **false**
```yaml
elasticsearch:
  url:
    scheme: "https"
    host: "127.0.0.1:9200"
  indices:
    visibility: cadence-visibility-dev
  tls:
    enabled: true
    caFile: /secrets/cadence/elasticsearch_cert.pem
    enableHostVerification: true
    serverName: myServerName
    certFile: /secrets/cadence/certfile.crt
    keyFile: /secrets/cadence/keyfile.key
    sslmode: false
```

## Local Testing

1. Increase Docker memory to higher than 6GB. Navigate to Docker -> Preferences -> Advanced -> Memory
2. Get the Cadence Docker compose file. Run `curl -O https://raw.githubusercontent.com/uber/cadence/master/docker/docker-compose-es.yml`
3. Start Cadence Docker (which contains Apache Kafka, Apache Zookeeper, and Elasticsearch) using `docker-compose -f docker-compose-es.yml up`
4. From the Docker output log, make sure Elasticsearch and Cadence started correctly. If you encounter an insufficient disk space error, try `docker system prune -a --volumes`
5. Register a local domain and start using it. `cadence --do samples-domain d re`
6. Allowlist search attributes. `cadence --do domain adm cl asa --search_attr_key NewKey --search_attr_type 1`

Note: starting a :workflow: with search attributes but without Elasticsearch will succeed as normal, but will not be searchable and will not be shown in list results.
