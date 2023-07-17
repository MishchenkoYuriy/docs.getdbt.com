---
title: "JDBC"
id: sl-jdbc
description: "Integrate and use the JDBC API to query your metrics."
tags: ["semantic-layer, apis"]
---

<VersionBlock lastVersion="1.5">

:::info Upgrade to access the new dbt Semantic Layer

The new dbt Semantic Layer has been re-released and is now available for users on a [Team or Enterprise plans](https://www.getdbt.com/pricing/) and you must be on dbt v1.6 and higher. 

If you're using the legacy Semantic Layer, we **highly** recommend you [upgrade your dbt version](/docs/dbt-versions/upgrade-core-in-cloud) to dbt v1.6 or higher to use the new dbt Semantic Layer. Refer to the dedicated [migration guide](/guides/migration/sl-migration) for more info.

:::

</VersionBlock>

The dbt Semantic Layer Java Database Connectivity (JDBC) API enables users to query metrics and dimensions using the JDBC protocol, while also providing standard metadata functionality. 

A JDBC driver is a software component enabling a Java application to interact with a data platform. Here's some more information about our JDBC API:

- The Semantic Layer JDBC API utilizes the open-source JDBC driver with ArrowFlight SQL protocol.
- You can download the JDBC driver from [Maven](https://search.maven.org/remotecontent?filepath=org/apache/arrow/flight-sql-jdbc-driver/12.0.0/flight-sql-jdbc-driver-12.0.0.jar). 
- The dbt Semantic Layer supports ArrowFlight SQL driver version 12.0.0 and higher. 
- You can embed the driver into your application stack as needed, and you can use dbt Labs' [example project](https://github.com/dbt-labs/example-semantic-layer-clients) for reference.
- If you’re a partner or user building a homegrown application, you’ll need to install an AWS root CA to the Java Trust [documentation](https://www.amazontrust.com/repository/) (specific to Java and JDBC call).

## Connection parameters

The JDBC connection requires a few different connection parameters. We provide the full JDBC string that you can connect with as well as the individual components required.

This is an example of a URL connection string and the individual components: 

```
jdbc:arrow-flight-sql://semantic-layer.cloud.getdbt.com:443?&environmentId=202339&token=SERVICE_TOKEN
```

| JDBC parameter | Description | Example |
| -------------- | ----------- | ------- |
| `jdbc:arrow-flight-sql://` | The protocol for the JDBC driver.  | `jdbc:arrow-flight-sql://` |
| `semantic-layer.cloud.getdbt.com` | The [access URL](/docs/cloud/about-cloud/regions-ip-addresses) for your account's dbt Cloud region. You must always add the `semantic-layer` prefix before the access URL.  | For dbt Cloud deployment hosted in North America, use `semantic-layer.cloud.getdbt.com`  |
| `environmentId` | The unique identifier for the dbt production environment, you can retrieve this from the dbt Cloud URL <br /> when you navigate to **Environments** under **Deploy**. | If your URL ends with `.../environments/222222`, your `environmentId` is `222222`<br /><br />   |
| `SERVICE_TOKEN` | dbt Cloud [service token](/docs/dbt-cloud-apis/service-tokens) with “Semantic Layer Only” permission. Create a new service token in your **Account Settings** page. Encode the value before inserting it into the string.  | `token=SERVICE_TOKEN` |

*Note &mdash; If you're testing locally on a tool like DataGrip, you may also have to provide the following variable at the end or beginning of the JDBC URL `&disableCertificateVerification=true`.

## Querying the API for metric metadata

The Semantic Layer JDBC API has built-in metadata calls which can provide a user with information about their metrics and dimensions. Here are some metadata commands and examples:

<Tabs>

<TabItem value="allmetrics" label="Fetch all defined metrics">

Use this query to fetch all defined metrics in your dbt project:

```bash
select * from {{ 
	semantic_layer.metrics() 
}}
```
</TabItem>

<TabItem value="alldimensions" label="Fetch all dimensions for a metric">

Use this query to fetch all dimensions for a metric. 

Note, `metrics` is a required argument that lists with one or multiple metrics in it.

```bash
select * from {{ 
	semantic_layer.dimensions(metrics=['food_order_amount'])}}
```


</TabItem>

<TabItem value="dimensionvalueformetrics" label="Fetch dimension values metrics">

Use this query to fetch dimension values for one or multiple metrics and single dimension. 

Note, `metrics` is a required argument that lists with one or multiple metrics in it, and a single dimension. 

```bash
select * from {{ 
semantic_layer.dimension_values(metrics=["food_order_amount"], group_by="customer__customer_name")}}
```

</TabItem>

</Tabs>

## Querying the API for metric values

To query metric values, here are the following parameters that are available:

| Parameter | Description  | Example    | Type |
| --------- | -----------| ------------ | -------------------- |
| `metrics`   | The metric name as defined in your dbt metric configuration   | `metrics=[revenue]` | Required    |
| `group_by`  | Dimension names or entities to group by. We require a reference to the entity of the dimension (other than for the primary time dimension), which is pre-appended to the front of the dimension name with a double underscore. | `group_by=[user__country, metric_time]`     | Optional   |
| `grain`   | A parameter specific to any time dimension and changes the grain of the data from the default for the metric. | ```group_by=[`Dimension('metric_time').``` <br/> ```grain('week\|day\|month\|quarter\|year')]``` | Optional     |
| `where`     | A where clause that allows you to filter on dimensions   | `where="metric_time >= '2022-03-08'"`   | Optional   |
| `limit`   | Limit the data returned    | `limit=10` | Optional  |
|`order`  | Order the data returned     | `order_by=['-order_gross_profit']` (remove `-` for ascending order)  | Optional   |
| `explain`   | If true, returns generated SQL for the data platform but does not execute | `explain=True`   | Optional |

## Examples

Use the following examples to help you get started with the JDBC API

### Fetch metadata for metrics

You can filter/add any SQL outside of the templating syntax. For example, you can use the following query to fetch the name and dimensions for a metric: 

```bash
select name, dimensions from {{ 
	semantic_layer.metrics() 
	}}
	WHERE name='food_order_amount'
``` 

### Query common dimensions

You can select common dimensions for multiple metrics. Use the following query to fetch the name and dimensions for multiple metrics: 

```bash
select * from {{ 
	semantic_layer.dimensions(metrics=['food_order_amount', 'order_gross_profit'])
	}}
``` 

### Query grouped by time

The following example query uses the [shorthand method](#faqs) to fetch revenue and new customers grouped by time:

```bash
select * from {{
	semantic_layer.query(metrics=['food_order_amount','order_gross_profit'], 
	group_by=['metric_time'])
	}}
``` 

### Query with a time grain

Use the following example query to fetch multiple metrics with a change time dimension granularities:

```bash
select * from {{
	semantic_layer.query(metrics=['food_order_amount', 'order_gross_profit'], 
	group_by=[Dimension('metric_time').grain('month')])
	}}
```

### Group by categorical dimension

Use the following query to group by a categorical dimension:

```bash
select * from {{
	semantic_layer.query(metrics=['food_order_amount', 'order_gross_profit'], 
	group_by=[Dimension('metric_time').grain('month'), 'customer__customer_type'])
	}}
``` 

### Query with a filter

Use the following example to query using a `where` filter:

```bash
select * from {{
semantic_layer.query(metrics=['food_order_amount', 'order_gross_profit'],
group_by=[Dimension('metric_time').grain('month'),'customer__customer_type'],
where="metric_time__month >= '2017-03-09' AND customer__customer_type in ('new')")
}}
``` 

### Query with a limit and order_by

Use the following example to query using a `limit` or `order_by` clauses:

```bash
select * from {{
semantic_layer.query(metrics=['food_order_amount', 'order_gross_profit'],
  group_by=[Dimension('metric_time')],
  limit=10,
  order_by=['order_gross_profit'])
  }}
``` 
### Query with explain keyword

Use the following example to query using a `explain` keyword:

```bash
select * from {{
semantic_layer.query(metrics=['food_order_amount', 'order_gross_profit'],
		group_by=[Dimension('metric_time').grain('month'),'customer__customer_type'],
		where="metric_time__month >= '2017-03-09' AND customer__customer_type in ('new')",
		explain=True)
		}}
```

## FAQs

- **Why do some dimensions use different syntax, like `metric_time` versus `[Dimension('metric_time')`?**<br />
	When you select a dimension on its own, such as `metric_time` you can use the shorthand method which doesn't need the “Dimension” syntax. However, when you perform operations on the dimension, such as adding granularity, the object syntax `[Dimension('metric_time')` is required. 

- **What does the double underscore `"__"` syntax in dimensions mean?**<br />
	The double underscore `"__"` syntax indicates a mapping from an entity to a dimension, as well as where the dimension is located. For example, `user__country` means someone is looking at the `country` dimension from the `user` table.

- **What is the default output when adding granularity?**<br />
	The default output follows the format `{time_dimension_name}__{granularity_level}`. So for example, if the time dimension name is `ds` and the granularity level is yearly, the output is `ds__year`.
