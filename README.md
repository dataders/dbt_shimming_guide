# Shimming dbt packages

## Background

### Adapter Disptach

I've been hands-on with dbt for about a year now, and as a maintainer of both the `dbt-sqlserver` and `dbt-synape` adapters. The `adapter.disptach()` functionality (introduced in `v0.18.0`) has allowed us to stand on the shoulders of giants by being able to adapt to our databases with minimal effort, the work of not only the dbt-core team but also that of the larger dbt community.

[More](https://docs.getdbt.com/docs/contributing/building-a-new-adapter/#adapter-dispatch) [info](https://docs.getdbt.com/reference/dbt-jinja-functions/adapter/#dispatch) on adapter dispatch

If you're interested in why shimming is recommended for  new adapter macros instead of upstream merges or forks, check out [this GitHub discussion from a few weeks ago](https://github.com/dbt-msft/tsql-utils/issues/36#issuecomment-809123275)

### dbt projects all the way down

Shim packages can contain shim macros for multiple packages. For example:
- `spark-utils` adds support for `dbt-utils` and `snowplow`, and
- `tsql-utils` adds support for `dbt-utils` and `dbt-date` packages.

The recommended approach here is to make the shim package import the parents repos as submodules. In this way
- the parent package's integration tests can be used to test that the shims are properly working without having to write new tests, and
- the end-user won't have all the parent packages as dependencies -- e.g. if a dbt-spark user wants to use `dbt-utils` but didn't want to install the `snowplow` package, they could do so.

The implication of this is that your shim package will many distinct `dbt_project.yml`'s. So if for example we want to shim dbt-utils to make it compatible with Firebolt db and we want to call the package: `firebolt-utils`, then these would be our dbt projects:

```shell
# firebolt-utils project
#   standard except require-dbt-version: ">=0.18.0"
/firebolt-utils/dbt_project.yml

# dbt-utils project, i.e. parent package
/firebolt-utils/dbt-utils/dbt_project.yml

# dbt-utils integration tests project
#   firebolt-utils will re-use dbt-utils's tests to
#   verify shim package shims correctly
/firebolt-utils/dbt-utils/integration_tests/dbt_project.yml

# where/how firebolt-utils will call dbt-util's integration tests
/firebolt-utils/integration_tests/dbt_utils/dbt_project.yml
```

This seems rather complicated, but if you exclude the projects from repos you don't own (i.e. dbt-utils -- the middle two), there are just two projects:
- the top-level project for the shim package, and
- the integration test project.

In grokking this, you achieve a new level of enlightenment at being able to behold another another facet of beauty of the dbt diamond:
> dbt packages are dbt projects and dbt projects are dbt packages

## Action Items

All of this multi-project infra given above is so that we can run dbt-utils's integration tests but also have:
- a profile with `type: firebolt`,
- `dbt-utils` and `firebolt-utils` macros installed,
- `dbt-utils` knows about the `firebolt-utils` macros, and
- control to tweak some of the test models where needed

To do this we:
1. ensure that all parent package macros correctly implement `adapter.dispatch`
1. create an `integration_tests` profile with `type: firebolt`
1. create the submodule and dbt projects as given above
2. Add the following `packages.json` to your `/firebolt-utils/integration_tests/dbt_utils` project
    ```
    packages:
    - local: ../../  # i.e. firebolt-utils
    - local: ../../dbt-utils
    - local: ../../dbt-utils/integration_tests
    ```
3. add the following variable to `/firebolt-utils/integration_tests/dbt_utils`'s `dbt_project.yml` in order to let `dbt-utils` knows about the `firebolt-utils` macros. Provided that parent package has a `_get_utils_namespaces()` macro.
    ```
    vars:
      dbt_utils_dispatch_list: ['firebolt_utils']
    ```
4. call the standard dbt commands until something inevitably breaks, while also watching for the gotchas given below
5. write shim macros accordingly until all parent package tests pass

## Gotchas
### keyword arguments

As a convenience, all jinja macros can accept additional keyword arguments without having to add `**kwargs` to the definition as you would in Python. The downside to this is that jinja macros will not look for keyword arguments unless they are mentioned in the body of the macro.

The hitch here (at least for tsql-utils which supports two different adapters) is that you have to invoke the `kwargs.get()`  at each macro def,  which is redundant but necessary.

```sql
{% macro sqlserver__test_not_constant(model) %}

    {% set column_name = kwargs.get('column_name', kwargs.get('arg')) %}

    select count(*) from (
        select
            count(distinct {{ column_name }}) as filler_column
        from {{ model }}
        having count(distinct {{ column_name }}) = 1
        ) validation_errors

{% endmacro %}

{% macro synapse__test_not_constant(model) %}
    {% set column_name = kwargs.get('column_name', kwargs.get('arg')) %}

    {% do return( tsql_utils.sqlserver__test_not_constant(model, **kwargs)) %}
{% endmacro %}

```
### over-riding parent package's integration test definitions

Sometimes the integration tests defined in the parent package includes an incompatible definition. An example is this [dbt-utils integration test](https://github.com/fishtown-analytics/dbt-utils/blob/bbba960726667abc66b42624f0d36bbb62c37593/integration_tests/models/schema_tests/schema.yml#L67-L75) where `<>` isn't supported in TSQL instead you must use `!=`.

```sql
  - name: data_test_relationships_where_table_2
    columns:
      - name: id
        tests:
          - dbt_utils.relationships_where:
              from: id
              to: ref('data_test_relationships_where_table_1')
              field: id
              from_condition: id <> 4
```

Rather than make a PR to dbt-utils changing `<>` to `!=`, which might break tests for other adapters, the solution is to catch when `<>` is used and replace it within your shimmed macro, as was done in [`sqlserver__test_relationships_where()`](https://github.com/dbt-msft/tsql-utils/blob/8d6154124aa30ce1c2afc2621432a6d43eea8bba/macros/dbt_utils/schema_tests/relationships_where.sql#L6-L11).

```
  {% if from_condition == 'id <> 4' %}
      {% set where = 'id != 4' %}
  {% endif %}
```

This isn't an ideal solution as this logic is executed every time a TSQL user invokes `dbt_utils.relationships_where()`, even though it only exists to pass the parent package integration test, but it's a tradeoff I'm happy with that we can properly test tsql-utils.
