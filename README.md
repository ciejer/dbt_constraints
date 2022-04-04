# dbt Constraints Package

This package generates database constraints based on the tests in a dbt project. It is currently compatible with Snowflake and PostgreSQL only. 

## Why data engineers should add referential integrity constraints

The primary reason to add constraints to your database tables is that many tools including [DBeaver](https://dbeaver.io) and [Oracle SQL Developer Data Modeler](https://community.snowflake.com/s/article/How-To-Customizing-Oracle-SQL-Developer-Data-Modeler-SDDM-to-Support-Snowflake-Variant) can correctly reverse-engineer data model diagrams if there are primary keys, unique keys, and foreign keys on tables. Most BI tools will also add joins automatically between tables when you import tables that have foreign keys. This can both save time and avoid mistakes.

In addition, although Snowflake doesn't enforce most constraints, the [query optimizer can consider primary key, unique key, and foreign key constraints](https://docs.snowflake.com/en/sql-reference/constraints-properties.html?#extended-constraint-properties) during query rewrite if the constraint is set to RELY. Since dbt can test that the data in the table complies with the constraints, this package creates constraints on Snowflake with the RELY property to improve query performance. 

Many other databases including PostgreSQL, SQL Server, Oracle, MySQL, and DB2 can use referencial integrity constraints to perform "[join elimination](https://blog.jooq.org/join-elimination-an-essential-optimiser-feature-for-advanced-sql-usage/)" to remove tables from an execution plan. This commonly occurs when you query a subset of columns from a view and some of the tables in the view are unnecessary. Even on databases that do not support join elimination, some [BI and visualization tools will also rewrite their queries](https://docs.snowflake.com/en/user-guide/table-considerations.html#referential-integrity-constraints) based on constraint information, producing the same effect.

Finally, although most columnar databases including Snowflake do not use or need indexes, most row-oriented databases including PostgreSQL require indexes on their primary key columns in order to perform efficient joins between tables. Typically a primary key or unique key constraint is enforced on such databases using such indexes. Having dbt create the unique indexes automatically can slightly reduce the degree of performance tuning necessary for row-oriented databases. Row-oriented databases frequently also need indexes on foreign key columns but [that is something best added manually](https://docs.getdbt.com/reference/resource-configs/postgres-configs#indexes).

## Please note

When you add this package, dbt will automatically begin to create unique keys for all your existing `unique` and `dbt_utils.unique_combination_of_columns` tests and foreign keys for existing `relationship` tests. The package also provides three new tests (`primary_key`, `unique_key`, and `foreign_key`) that are a bit more flexible than the standard dbt tests. These tests can be used inline, out-of-line, and can support multiple columns when used in the `tests:` section of a model.

### Disabling automatic constraint generation

The `dbt_constraints_enabled` variable can be set to `false` in your project to disable automatic constraint generation.

```yml
vars:
  dbt_constraints_enabled: false
```

## Installation

1. Add this package to your `packages.yml` following [these instructions](https://docs.getdbt.com/docs/building-a-dbt-project/package-management/). If you are comfortable with testing the very latest code, the following code will pull the very latest version of the package.
```yml
packages:
  - git: "https://github.com/danflippo/dbt_constraints.git"
    revision: main
```

2. Run `dbt deps`.

3. Optionally add `primary_key`, `unique_key`, or `foreign_key` tests to your model like the following examples.
```yml
  - name: DIM_ORDER_LINES
    columns:
      # Single column inline constraints
      - name: OL_PK
        tests:
          - dbt_constraints.primary_key
      - name: OL_UK
        tests:
          - dbt_constraints.unique_key
      - name: OL_CUSTKEY
        tests:
          - dbt_constraints.foreign_key:
              pk_table_name: ref('DIM_CUSTOMERS')
              pk_column_name: C_CUSTKEY
    tests:
      # Single column constraints
      - dbt_constraints.primary_key:
          column_name: OL_PK
      - dbt_constraints.unique_key:
          column_name: OL_ORDERKEY
      - dbt_constraints.foreign_key:
          fk_column_name: OL_CUSTKEY
          pk_table_name: ref('DIM_CUSTOMERS')
          pk_column_name: C_CUSTKEY
      # Multiple column constraints
      - dbt_constraints.primary_key:
          column_names:
            - OL_PK_COLUMN_1
            - OL_PK_COLUMN_2
      - dbt_constraints.unique_key:
          column_names:
            - OL_UK_COLUMN_1
            - OL_UK_COLUMN_2
      - dbt_constraints.foreign_key:
          fk_column_names:
            - OL_FK_COLUMN_1
            - OL_FK_COLUMN_2
          pk_table_name: ref('DIM_CUSTOMERS')
          pk_column_names:
            - C_PK_COLUMN_1
            - C_PK_COLUMN_2
```

### Dependencies and Requirements

* The package's macros depend on the results and graph object schemas of dbt >=1.0.0

* The package currently only includes macros for creating constraints in Snowflake and PostgreSQL. To add support for other databases, it is necessary to implement the following five macros with the appropriate DDL & SQL for your database. Pull requests to contribute support for other databases are welcome. See the snowflake__create_constraints.sql and postgres__create_constraints.sql files as examples.

```
<ADAPTER_NAME>__create_primary_key(table_model, column_names, quote_columns=false)
<ADAPTER_NAME>__create_unique_key(table_model, column_names, quote_columns=false) 
<ADAPTER_NAME>__create_foreign_key(test_model, pk_model, pk_column_names, fk_model, fk_column_names, quote_columns=false) 
<ADAPTER_NAME>__unique_constraint_exists(table_relation, column_names) 
<ADAPTER_NAME>__foreign_key_exists(table_relation, column_names) 
```

## dbt_constraints Limitations

Generally, if you don't meet a requirement, tests are still executed but the constraint is skipped rather than producing an error.

- All models involved in a constraint must be materialized as table, incremental, or snapshot

- Constraints will not be created on sources, only models. You can use the PK/UK/FK tests with sources but constraints won't be generated.

- All columns on constraints must be individual column names, not expressions. You can reference columns on a model that come from an expression.

- Constraints are not created for failed tests

- `primary_key`, `unique_key`, and `foreign_key` tests are considered first and duplicate constraints are skipped. One exception is that you will get an error if you add two different `primary_key` tests to the same model.

- Foreign keys require that the parent table have a primary key or unique key on the referenced columns. Unique keys generated from standard `unique` tests are sufficient.

- The order of columns on a foreign key test must match betweek the FK columnns and PK columns

- The `foreign_key` test will ignore any rows with a null column, even if only one of two columns in a compound key is null. If you also want to ensure FK columns are not null, you should add standard `not_null` tests to your model.
