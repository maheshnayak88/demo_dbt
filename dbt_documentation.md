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
  - [Data Documentation with dbt CLI](#data-documentation-with-dbt-cli)

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
    FROM {{ ref('source_orders') }}
),

processed_orders AS (
    SELECT
        order_id,
        order_date,
        customer_id,
        order_status,
        SUM(order_item_quantity * order_item_price) AS total_revenue
    FROM raw_orders
    GROUP BY 1, 2, 3, 4
)

SELECT * FROM processed_orders
```

![VSCode - Create Model](https://i.imgur.com/5D5HmZY.png)

### Running Your Model

To run your model, open the terminal in VSCode and run the following command:

```bash
dbt run
```

![VSCode - Run Model](https://i.imgur.com/5qsO8kE.png)

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
  - name: orders
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
```

2. Data test (in a new `tests` directory):

Create a new file named `check_order_revenue.sql` inside the `tests` directory and add the following SQL code:

```sql
SELECT
    order_id,
    total_revenue
FROM {{ ref('orders') }}
WHERE total_revenue < 0
```

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

![dbt Lineage](https://i.imgur.com/yKUyQyp.png)

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

dbt Utils is a package of utility macros that can help you write more efficient and concise code. To install the package, add the following configuration to your `dbt_project.yml` file:

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: 0.7.0
```

Then, run `dbt deps` to install the package.

To use a dbt Utils function, simply call it in your model using the `{% ... %}` syntax. For example:

```sql
SELECT
    order_id,
    customer_id,
    {% dbt_utils.surrogate_key(order_id, customer_id) %} as unique_key
FROM {{ ref('orders') }}
```

### Incremental Models

Incremental models allow you to update only new or changed data, making them more efficient for large datasets. To create an incremental model, set the materialization option in your `dbt_project.yml` file as shown earlier. Then, add a filter in your model to select only the new or updated data. For example:

```sql
{% if is_incremental() %}
    WHERE order_date >= (SELECT MAX(order_date) FROM {{ this }})
{% endif %}
```

### Data Freshness

Data freshness is a metric that measures how up-to-date your data is-

```
in your data warehouse. To configure data freshness in your dbt project, add a `freshness` key to your `sources.yml` file. For example:

```yaml
version: 2

sources:
  - name: raw_data
    tables:
      - name: source_orders
        identifier: "raw_orders"
        freshness:
          warn_after:
            count: 48
            period: hour
          error_after:
            count: 72
            period: hour
```

### Generating Data Documentation

You can generate data documentation for your dbt project using the dbt CLI. To do this, run the following command:

```bash
dbt docs generate
```

This command will generate a static HTML site with your project documentation. To view the documentation, run:

```bash
dbt docs serve
```

This will open the documentation site in your web browser.

![dbt Docs](https://i.imgur.com/CTTgTJt.png)

## Recap

We've covered a lot of ground in this dbt documentation! By now, you should have a solid understanding of dbt core concepts, including models, tests, macros, materialization, and more. You've also learned how to use VSCode as your IDE, working with GitLab for source control, and taking advantage of dbt Utils for common functions.

Remember to take advantage of the extensive dbt [documentation](https://docs.getdbt.com/) and [community](https://community.getdbt.com/) as you continue to build your data transformation pipelines.

Happy data modeling!