## dbt learning
### Setup
- set up **dbt** using the [installation guide](https://github.com/cityblock/mixer/tree/master/dbt#installation) found in `mixer/dbt`
- use the `dbt_env` conda environment
- Installed the **Better Jinja** VSCode extension and select `Jinja SQL` as the syntax highlighter

### Tutorial
- following along with the dbt tutorial [here](https://docs.getdbt.com/tutorial/setting-up)
    - `dbt init {project-name}` allows you to initialize a new dbt project scaffold; make `{project-name}` the same as the git repo you're using
    ```shell
    .
    ├── README.md
    ├── analysis
    ├── data
    ├── dbt_project.yml
    ├── macros
    ├── models
    │   └── example
    │       ├── my_first_dbt_model.sql
    │       ├── my_second_dbt_model.sql
    │       └── schema.yml
    ├── snapshots
    └── tests

    7 directories, 5 files
    ```
- I am using the default `dsa` profile in `~/.dbt/profiles.yml` for the `profile` in `dbt_project.yml`
- Run `dbt debug` to debug projects
- After first `dbt run`:
```shell
.
├── README.md
├── analysis
├── data
├── dbt_modules
├── dbt_project.yml
├── logs
│   └── dbt.log
├── macros
├── models
│   └── example
│       ├── my_first_dbt_model.sql
│       ├── my_second_dbt_model.sql
│       └── schema.yml
├── snapshots
├── target
│   ├── compiled
│   │   └── jaffle_shop
│   │       └── models
│   │           └── example
│   │               ├── my_first_dbt_model.sql
│   │               └── my_second_dbt_model.sql
│   ├── graph.gpickle
│   ├── manifest.json
│   ├── partial_parse.msgpack
│   ├── run
│   │   └── jaffle_shop
│   │       └── models
│   │           └── example
│   │               ├── my_first_dbt_model.sql
│   │               └── my_second_dbt_model.sql
│   └── run_results.json
└── tests

18 directories, 14 files
```
- See results of run in BigQuery under my project (`cbh-reed-mcdaniel`) in the `dbt_reed_mcdaniel` scratch dataset
    - views and tables are nested underneath that
- Rewriting the following SQL query into a dbt model in `models/customers.sql`:
```sql
with customers as (

    select
        id as customer_id,
        first_name,
        last_name

    from `dbt-tutorial`.jaffle_shop.customers

),

orders as (

    select
        id as order_id,
        user_id as customer_id,
        order_date,
        status

    from `dbt-tutorial`.jaffle_shop.orders

),

customer_orders as (

    select
        customer_id,

        min(order_date) as first_order_date,
        max(order_date) as most_recent_order_date,
        count(order_id) as number_of_orders

    from orders

    group by 1

),


final as (

    select
        customers.customer_id,
        customers.first_name,
        customers.last_name,
        customer_orders.first_order_date,
        customer_orders.most_recent_order_date,
        coalesce(customer_orders.number_of_orders, 0) as number_of_orders

    from customers

    left join customer_orders using (customer_id)

)

select * from final
```
#### Materialization
- you can easily configure whether a SQL statement materializes as a `view` or a `table` in dbt without explicit SQl commands (by defualt, they're all configured as `view`s)
- this can be done one of two ways
    1. In the `dbt_project.yml` under `models`
    2. In the model files themselves using the `{{ config(materialized={"view" or "table"}) }}` macro
        - ***Note: the macros in the model files take precedence and overwite what is specified in `dbt_project.yml`***
- ***To go from a `view` to a `table` is no issue, but to go from a `table` to a `view` you must run `dbt run --full-refresh` for that change to take effect with no errors***

---
- In `target/compile` and `target/run` you can get more details on changes compiled and run
- See `logs` for detailed logs on all operations taking place under the hood
- ***Deleted example models and references in `dbt_project.yml` > must then manually delete the relations from BigQuery***

---
#### Compound Models
- CTEs = Common Table Expressions = A CTE (common table expression) is a named subquery defined in a `WITH` clause (can be thought of as a temporary `view`) (more info [here](https://docs.snowflake.com/en/user-guide/queries-cte.html#what-is-a-cte))
    - We want to be able to separate logic that pre-processes and cleans data (i.e. renaming columns, creating CTEs, etc.), from logic that transforms data
    - files prefixed with `stg_` are files for "staging" or the pre-processing of the data
    - then in the other files, you can reference the pre-processed data with the `{{ ref('{model name}') }}` macro
    - dbt can infer the order in which the models must be created based on dependency of the `ref` macro
    - cna look at `target/compiled/jaffe_shop/customers.sql` to see the how dbt sources the actually table reference when interpreting the `ref` so that it makes sense to BigQuery
    - moved staging models into their own `models/staging` directory, then set them to be materialized as `view`s in `dbt_project.yml`
    - see the [syntax guide](https://docs.getdbt.com/reference/node-selection/syntax) for how to run only some models: `dbt run --models staging`

---
#### Testing and Documentation
- `schema.yml` file (name can change but must be a YAML file) contains schema definitions that can be tested
- running `dbt test` will check the definitions and return a number greater than 1 if it fails i.e. 0 for success
- you can find the compiled logic for these tests in `target/compiled/jaffle_shop/models/schema.yml/schema_test`
- the `not_null`, `unique`, and `relationship` tests come with dbt out of the box; we can also create our own custom test logic
- documentation is achieved using the `description` YAML key in files
- documentation can then be generated by running `dbt docs generate`, the output of which can be found in `target/catalog.json`
    - run `dbt docs serve` to see the docs served at `localhost:8080` with cool dependency graph (think UML diagram)

### Resources:
- Learn more about dbt [in the docs](https://docs.getdbt.com/docs/introduction)
- Check out [Discourse](https://discourse.getdbt.com/) for commonly asked questions and answers
- Join the [chat](http://slack.getdbt.com/) on Slack for live discussions and support
- Find [dbt events](https://events.getdbt.com) near you
- Check out [the blog](https://blog.getdbt.com/) for the latest news on dbt's development and best practices
