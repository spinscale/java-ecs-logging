# Log4j2 ECS Layout

## Step 1: add dependency

Latest version: [![Maven Central](https://img.shields.io/maven-central/v/co.elastic.logging/log4j2-ecs-layout.svg)](https://search.maven.org/search?q=g:co.elastic.logging%20AND%20a:log4j2-ecs-layout)

Add a dependency to your application
```xml
<dependency>
    <groupId>co.elastic.logging</groupId>
    <artifactId>log4j2-ecs-layout</artifactId>
    <version>${java-ecs-logging.version}</version>
</dependency>
```

## Step 2: use the `EcsLayout`

Instead of the usual `<PatternLayout/>`, use `<EcsLayout serviceName="my-app"/>`.

## Example
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="DEBUG">
    <Appenders>
        <Console name="LogToConsole" target="SYSTEM_OUT">
            <EcsLayout serviceName="my-app"/>
        </Console>
        <File name="LogToFile" fileName="logs/app.log">
            <EcsLayout serviceName="my-app"/>
        </File>
    </Appenders>
    <Loggers>
        <Root level="info">
            <AppenderRef ref="LogToFile"/>
            <AppenderRef ref="LogToConsole"/>
        </Root>
    </Loggers>
</Configuration>
```

## Layout Parameters

|Parameter name   |Type   |Default|Description|
|-----------------|-------|-------|-----------|
|serviceName      |String |       |Sets the `service.name` field so you can filter your logs by a particular service |
|topLevelLabels   |String |`trace.id, transaction.id, span.id, error.id, service.name`|Usually, MDC keys are nested under [`labels`](https://www.elastic.co/guide/en/ecs/current/ecs-base.html). You can specify a comma-separated list of properties which should be on the top level. |
|includeMarkers   |boolean|`false`|Log [Markers](https://logging.apache.org/log4j/2.0/manual/markers.html) as [`tags`](https://www.elastic.co/guide/en/ecs/current/ecs-base.html) |
|stackTraceAsArray|boolean|`false`|Serializes the [`error.stack_trace`](https://www.elastic.co/guide/en/ecs/current/ecs-error.html) as a JSON array where each element is in a new line to improve readability. Note that this requires a slightly more complex [Filebeat configuration](../README.md#when-stacktraceasarray-is-enabled).|
|includeOrigin    |boolean|`false`|If `true`, adds the [`log.origin.file.name`](https://www.elastic.co/guide/en/ecs/current/ecs-log.html), [`log.origin.file.line`](https://www.elastic.co/guide/en/ecs/current/ecs-log.html) and [`log.origin.function`](https://www.elastic.co/guide/en/ecs/current/ecs-log.html) fields. Note that you also have to set `includeLocation="true"` on your loggers and appenders if you are using the async ones. |

To include any custom field in the output, use following syntax:

```xml
  <EcsLayout>
    <KeyValuePair key="additionalField1" value="constant value"/>
    <KeyValuePair key="additionalField2" value="$${ctx:key}"/>
  </EcsLayout>
```

Custom fields are included in the order they are declared. The values support [lookups](https://logging.apache.org/log4j/2.x/manual/lookups.html).
## Structured logging

By leveraging log4j2's `MapMessage` or even by implementing your own `MultiformatMessage` with JSON support,
you can add additional fields to the resulting JSON.

Example:

```java
logger.info(new StringMapMessage().with("message", "foo").with("foo", "bar"));
``` 

### Gotchas

A common pitfall is how dots in field names are handled in Elasticsearch and how they affect the mapping.
In recent Elasticsearch versions, the following JSON structures would result in the same index mapping:

```json
{
  "foo.bar": "baz"
}
```

```json
{
  "foo": {
    "bar": "baz"
  }
}
```
The property `foo` would be mapped to the [Object datatype](https://www.elastic.co/guide/en/elasticsearch/reference/current/object.html).

This means that you can't index a document where `foo` would be a different datatype, as in shown in the following example:

```json
{
  "foo": "bar"
}
```

In that example, `foo` is a string.
Trying to index that document results in an error because the data type of `foo` can't be object and string at the same time.
 