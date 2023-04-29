# dbt Documentation 

Welcome to the exciting world of dbt (Data Build Tool)! This guide is designed for SQL-savvy analysts who are new to dbt and want to learn how to use it with BigQuery as a warehouse. We'll walk you through the steps to set up a dbt project, create your first model, run tests, and much more. Along the way, you'll see plenty of screenshots from VSCode to make things crystal clear.

## Table of Contents

- [Getting Started](#getting-started)
  - [Installation](#installation)
  - [Project Setup](#project-setup)
- [Your First Model](#your-first-model)
  - [What is a Model?](#what-is-a-model)
  - [Creating a Model](#creating-a-model)
  - [Running Your Model](#running-your-model)
  - [Model Selection](#model-selection)
- [Tests](#tests)
  - [Types of Tests](#types-of-tests)
  - [Creating Tests](#creating-tests)
- [Macros](#macros)
- [Project Structure](#project-structure)
- [Jinja and Lineage](#jinja-and-lineage)
- [Schema.yml](#schemayml)
  - [Best Practices](#best-practices)
  - [Adding Documentation](#adding-documentation)
  - [Adding Tests](#adding-tests)
- [Sources.yml](#sourcesyml)
  - [Sourcing Your First Model](#sourcing-your-first-model)
- [Model Parameters](#model-parameters)
- [Materialization](#materialization)
  - [Materialization Options](#materialization-options)
  - [Creating Materialized Models](#creating-materialized-models)
- [Advanced Topics](#advanced-topics)
  - [dbt Utils](#dbt-utils)
  - [Incremental Models](#incremental-models)
  - [Data Freshness](#data-freshness)
  - [Generating Data Documentation](#generating-data-documentation)

## Getting Started

### Installation

1. Install dbt following the instructions [here](https://docs.getdbt.com/dbt-cli/installation).
2. Set up VSCode as your IDE. Download and install it from [here](https://code.visualstudio.com/).

### Project Setup

1. Open VSCode and open the terminal (View > Terminal).
2. Run `dbt init my_bigquery_project` to create a new dbt project called "my_bigquery_project"
3. Open the new project folder in VSCode (File > Open Folder...).

![image](https://user-images.githubusercontent.com/88837021/235291849-d68cf696-f74a-4d89-9e49-fffa2f6ff139.png)

4. Configure your BigQuery profile:
   - Create a new file named `profiles.yml` in the `~/.dbt/` directory (or `%USERPROFILE%/.dbt/` on Windows).
   - Add the following configuration to the `profiles.yml` file, replacing the placeholders with your BigQuery account information:

```yaml
my_bigquery_project:
  target: dev
  outputs:
    dev:
      type: bigquery
      method: oauth
      project: <your_bigquery_project_id>
      dataset: <your_bigquery_dataset>
      threads: 1
      timeout_seconds: 300
      location: US
      priority: interactive
      retries: 1
```

For more details on BigQuery profile configurations, check [here](https://docs.getdbt.com/reference/warehouse-profiles/bigquery-profile).

5. In VSCode, create a new branch for your dbt project (Git
> Click on the branch name in the lower-left corner of VSCode
> Click "Create new branch" and enter a name, such as "feature/first_model"

![image](https://user-images.githubusercontent.com/88837021/235292401-080ecb92-e19f-416b-bdd1-d7bcf9ec335b.png)

## Your First Model

### What is a Model?

A model in dbt is a SQL file that defines a transformation, aggregation, or any other operation that creates a new dataset from existing data. Models are the building blocks of your analytics codebase and are used to construct your data warehouse.

### Creating a Model

1. In your project directory, create a new directory called `models` if it doesn't already exist.
2. Create a new file named `orders.sql` inside the `models` directory.
3. Add your SQL code to the `orders.sql` file. For example:

```sql
WITH raw_orders AS (
    SELECT *
    FROM {{ source('raw_data', 'source_orders') }}
),

clean_orders AS (
    SELECT
        order_id,
        customer_id,
        order_date,
        status,
        total_amount
    FROM raw_orders
    WHERE status <> 'canceled'
)

SELECT * FROM clean_orders
```
![image](https://user-images.githubusercontent.com/88837021/235298384-140e7336-22a4-497f-b785-78c8302fdd4c.png)


### Running Your Model

To run your model, open the terminal in VSCode and run the following command:

```bash
dbt run
```

![image](https://user-images.githubusercontent.com/88837021/235298347-43710a93-1326-49d2-8e82-2be3feff92dc.png)

### Model Selection

You can use selection methods to run specific models or groups of models. Some examples include:

- Run only the "orders" model:

```bash
dbt run --models orders
```

- Run all models in the "core" directory:

```bash
dbt run --models core.*
```

- Run only the models that are ancestors (upstream) of the "orders" model:

```bash
dbt run --models +orders
```

For more selection methods, check [here](https://docs.getdbt.com/reference/node-selection/syntax).

## Tests

### Types of Tests

There are three types of tests in dbt:

1. Schema tests: Validate that your data meets specific constraints, such as uniqueness or not null.
2. Data tests: Custom tests written in SQL to validate complex business logic or data quality rules.
3. Snapshot tests: Validate that historical data is accurate and changes are tracked correctly.

### Creating Tests

For each type of test, you'll need to create a new file and add the appropriate configuration or SQL code. Here's an example for each type:

1. Schema test (in your `schema.yml` file):

```yaml
version: 2

models:
  - name: clean_orders
    description: "Cleaned orders, excluding canceled orders"
    columns:
      - name: order_id
        description: "Unique identifier for the order"
        tests:
          - unique
          - not_null
      - name: customer_id
        description: "Unique identifier for the customer who placed the order"
      - name: order_date
        description: "Date when the order was placed"
        tests:
          - not_null
      - name: status
        description: "Status of the order"
      - name: total_amount
        description: "Total amount of the order"
        tests:
          - not_null
```

2. Data test (in a new `tests` directory):

Create a new file named `check_order_revenue.sql` inside the `tests/singular_tests` directory and add the following SQL code:

```sql
SELECT
    order_id,
    total_amount
FROM {{ ref('clean_orders') }}
WHERE total_amount < 0
```
![image](https://user-images.githubusercontent.com/88837021/235298670-cdfba629-977d-469e-934f-32b7a3eba77a.png)


3. Snapshot test (in your `snapshots` directory):

Create a new file named `snapshot_orders.sql` inside the `snapshots` directory and add the following SQL code:

```sql
{% snapshot snapshot_orders %}

SELECT *
FROM {{ ref('orders') }}

{% endsnapshot %}
```

Then, in your `schema.yml`
```
file, add a snapshot test configuration:

```yaml
snapshots:
  - name: snapshot_orders
    columns:
      - name: _dbt_valid_from
        tests:
          - not_null
```

## Macros

Macros are reusable pieces of Jinja code that can be used in your dbt project. To create a macro:

1. In your project directory, create a new directory called `macros` if it doesn't already exist.
2. Create a new file named `currency_conversion.sql` inside the `macros` directory.
3. Add your Jinja code to the `currency_conversion.sql` file. For example:

```jinja
{% macro convert_currency(amount, rate) %}
    {{ amount }} * {{ rate }}
{% endmacro %}
```

4. Use the macro in your model by calling it with the `{% ... %}` syntax. For example:

```sql
SELECT
    order_id,
    customer_id,
    {% convert_currency(total_revenue, 0.8) %} as converted_revenue
FROM {{ ref('orders') }}
```

Now let's create another macro as an example:

In your dbt project directory, create a new folder called macros.
Inside the macros folder, create a new file with the extension .sql for your macro, such as date_diff.sql.
Define your macro using the {% macro %} tag. For example:
```sql
{% macro date_diff(date_col1, date_col2) %}
    DATEDIFF('day', {{ date_col1 }}, {{ date_col2 }})
{% endmacro %}
```
In this example, we're creating a macro named date_diff that takes two arguments, date_col1 and date_col2, and returns the difference in days between the two dates.

Now, let's use the macro in a new model called customer_age.sql:

```sql
WITH customers AS (
    SELECT *
    FROM {{ source('raw_data', 'source_customers') }}
),

customer_age AS (
    SELECT
        customer_id,
        first_name,
        last_name,
        email,
        created_at,
        {{ date_diff('created_at', 'CURRENT_DATE()') }} as age_in_days
    FROM customers
)
SELECT * FROM customer_age
```

Finally, create a new entry in your schema.yml file for the customer_age model:

```yaml
  - name: customer_age
    description: "Customers with their age in days"
    columns:
      - name: customer_id
        description: "Unique identifier for the customer"
        tests:
          - unique
          - not_null
      - name: first_name
        description: "Customer's first name"
      - name: last_name
        description: "Customer's last name"
      - name: email
        description: "Customer's email address"
        tests:
          - not_null
      - name: created_at
        description: "Date when the customer was created"
        tests:
          - not_null
      - name: age_in_days
        description: "Customer's age in days"
        tests:
          - not_null

```
![VSCode - Create Macro](https://i.imgur.com/6ZkA2mY.png)

## Project Structure

A typical dbt project structure should look like this:

- `models/`: Contains your SQL model files.
- `tests/`: Contains your custom data tests.
- `snapshots/`: Contains your snapshot files.
- `macros/`: Contains your macro files.
- `analysis/`: Contains ad-hoc analysis files that aren't deployed to production.
- `dbt_project.yml`: Configures your project settings.
- `schema.yml`: Contains schema tests and documentation for your models.
- `sources.yml`: Contains source table configurations.

## Jinja and Lineage

Jinja is a templating language that allows you to write dynamic SQL code in dbt. It enables you to use control structures, variables, and macros to create more flexible and reusable models. Lineage refers to the dependencies between models and other objects in your project. dbt automatically tracks lineage, which allows you to understand the relationships between models and helps with testing and deployment.


## Schema.yml

The `schema.yml` file is used to define tests, documentation, and other metadata for your models. It's a key part of your dbt project and should be updated as you create and modify models.

### Best Practices

- Add tests for every model and column, such as uniqueness, not null, and referential integrity.
- Document every table and column, including a description, data type, and any other relevant information.
- Keep your `schema.yml` file organized by grouping models and columns logically.
- Use the `schema.yml` file to define relationships between tables, such as foreign key constraints.

### Adding Documentation

To add documentation to your tables and columns, include a `description` key in your `schema.yml` file. For example:

```yaml
version: 2

models:
  - name: orders
    description: "Processed orders with total revenue"
    columns:
      - name: order_id
        description: "Unique identifier for each order"
        tests:
          - unique
          - not_null
```

### Adding Tests

To add tests to your models and columns, include a `tests` key in your `schema.yml` file. For example:

```yaml
version: 2

models:
  - name: orders
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      - name: customer_id
        tests:
 
 
 
 Certainly! Here is the remaining documentation:

```
not_null
```

## Sources.yml

The `sources.yml` file is used to define source tables in your data warehouse, which are then used as inputs for your dbt models.

### Sourcing Your First Model

1. In your project directory, create a new file named `sources.yml`.
2. Add your source table configuration to the `sources.yml` file. For example:

```yaml
version: 2

sources:
  - name: raw_data
    tables:
      - name: source_orders
        identifier: "raw_orders"
```

3. Use the `ref()` function in your model to reference the source table. For example:

```sql
WITH raw_orders AS (
    SELECT *
    FROM {{ ref('source_orders') }}
),

... (rest of your model)
```

## Model Parameters

You can apply parameters to models individually in the `dbt_project.yml` file. For example, you can set the materialization for a specific model like this:

```yaml
models:
  my_bigquery_project:
    orders:
      materialized: table
```

## Materialization

Materialization is the process of converting your dbt model into a table, view, or other object in your data warehouse. There are several materialization options, such as `table`, `view`, `incremental`, and `ephemeral`.

### Materialization Options

- `table`: Creates a table from your model and truncates it before each run.
- `view`: Creates a view from your model. The data is not stored separately and is computed when the view is queried.
- `incremental`: Creates a table and only updates new or changed data, making it more efficient for large datasets.
- `ephemeral`: Creates a temporary table or subquery that is not persisted in the data warehouse.

For more details on materialization options, check [here](https://docs.getdbt.com/docs/building-a-dbt-project/building-models/materializations).

### Creating Materialized Models

To create a materialized model, set the materialization option in your `dbt_project.yml` file. For example, to create an incremental model:

```yaml
models:
  my_bigquery_project:
    orders:
      materialized: incremental
      incremental_strategy: merge
      unique_key: order_id
```

## Advanced Topics

### dbt Utils

Let's move on to the advanced section.

### Advanced

#### Installing and Using dbt-utils

`dbt-utils` is a package that contains several useful utility macros and functions for use in your dbt project. To install the `dbt-utils` package, follow these steps:

1. In your dbt project directory, open the `packages.yml` file. If it doesn't exist, create it.
2. Add the following lines to the `packages.yml` file:

```yaml
packages:
  - package: fishtown-analytics/dbt_utils
    version: 0.7.3
```

3. Run the following command to install the package:

```
dbt deps
```

Now, you can use the functions provided by `dbt-utils`. For example, let's create a model that uses the `surrogate_key` function to create a unique identifier for each row in the `customer_orders` model.

Create a new model called `customer_orders_surrogate_key.sql`:

```sql
{% set customer_order_columns = ['customer_id', 'order_id', 'first_name', 'last_name', 'email', 'order_date', 'total_amount'] %}

WITH customer_orders AS (
    SELECT *
    FROM {{ ref('customer_orders') }}
),

customer_orders_surrogate_key AS (
    SELECT
        {{ dbt_utils.surrogate_key(customer_order_columns) }} as unique_id,
        *
    FROM customer_orders
)

SELECT * FROM customer_orders_surrogate_key
```

Add a new entry in your `schema.yml` file for the `customer_orders_surrogate_key` model:

```yaml
  - name: customer_orders_surrogate_key
    description: "Customer orders with a surrogate key"
    columns:
      - name: unique_id
        description: "Unique identifier for each row"
        tests:
          - unique
          - not_null
      - name: customer_id
        description: "Unique identifier for the customer"
        tests:
          - unique
          - not_null
      - name: order_id
        description: "Unique identifier for the order"
        tests:
          - unique
          - not_null
      - name: first_name
        description: "Customer's first name"
      - name: last_name
        description: "Customer's last name"
      - name: email
        description: "Customer's email address"
        tests:
          - not_null
      - name: order_date
        description: "Date when the order was placed"
        tests:
          - not_null
      - name: total_amount
        description: "Total amount of the order"
```

Run your new model and its tests using the same dbt commands:

1. Run your models with `dbt run`.
2. Run your tests with `dbt test`.

#### Incremental Models

Incremental models are a way to speed up your dbt run by only processing new or changed data since the last run. To create an incremental model, you need to change the `materialized` property to `incremental` and include a filter for new data in your model.

For example, let's create an incremental version of the `clean_orders` model called `clean_orders_incremental.sql`:

```sql
{{ config(materialized='incremental') }}

WITH raw_orders AS (
    SELECT *
    FROM {{ source('raw_data', 'source_orders') }}
),

clean_orders AS (
    SELECT
        order_id,
        customer_id,
        order_date,
        status,
        total_amount
    FROM raw_orders
    WHERE status <> 'canceled'
)

SELECT * FROM clean_orders

{% if is_incremental() %}
  WHERE order_date > (SELECT MAX(order_date)
  FROM {{ this }})
{% endif %}

FROM {{ this }})
{% endif %}
```

Add a new entry in your `schema.yml` file for the `clean_orders_incremental` model:

```yaml
  - name: clean_orders_incremental
    description: "Cleaned orders, excluding canceled orders (incremental)"
    columns:
      - name: order_id
        description: "Unique identifier for the order"
        tests:
          - unique
          - not_null
      - name: customer_id
        description: "Unique identifier for the customer who placed the order"
      - name: order_date
        description: "Date when the order was placed"
        tests:
          - not_null
      - name: status
        description: "Status of the order"
      - name: total_amount
        description: "Total amount of the order"
        tests:
          - not_null
```

Run your new incremental model and its tests using the same dbt commands:

1. Run your models with `dbt run`.
2. Run your tests with `dbt test`.

#### Data Freshness

Data freshness is a feature that helps you track the age of the data in your models. To declare data freshness, you can add the `freshness` property to your `sources.yml` file:

```yaml
version: 2

sources:
  - name: raw_data
    database: your_snowflake_database
    schema: MY_SCHEMA
    freshness:
      warn_after: { count: 72, period: "hour" }
      error_after: { count: 168, period: "hour" }
    tables:
      - name: source_orders
      - name: source_customers
```

Now, you can run the `dbt source freshness` command to check the freshness of your sources:

```
dbt source freshness
```

#### Generating Data Documentation

You can generate data documentation for your dbt project using the dbt CLI. To do this, run the following command:

```bash
dbt docs generate
```

This command will generate a static HTML site with your project documentation. To view the documentation, run:

```bash
dbt docs serve
```
This will open the generated documentation in your default web browser, allowing you to explore your dbt project, models, sources, tests, and more.

Remember to take as many screenshots as possible from VSCode, especially when running commands and showing logs, to help your target audience understand the process better.

That's it! With this information, you should have a solid understanding of how to use dbt with Snowflake, create models, sources, tests, macros, and more.
This will open the documentation site in your web browser.

![image](https://user-images.githubusercontent.com/88837021/235299120-3176491c-63a2-41e0-9b3e-fd2ac95028a0.png)

## Recap

We've covered a lot of ground in this dbt documentation! By now, you should have a solid understanding of dbt core concepts, including models, tests, macros, materialization, and more. You've also learned how to use VSCode as your IDE, working with GitLab for source control, and taking advantage of dbt Utils for common functions.

Remember to take advantage of the extensive dbt [documentation](https://docs.getdbt.com/) and [community](https://community.getdbt.com/) as you continue to build your data transformation pipelines.

Happy data modeling!
