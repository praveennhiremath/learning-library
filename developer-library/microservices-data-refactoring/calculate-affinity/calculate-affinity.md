# Determine the affinity between tables

## Introduction

*Describe the lab in one or two sentences, for example:* This lab walks you through the steps to ...

Estimated Lab Time: -- minutes

### About <Product/Technology> (Optional)
Enter background information here about the technology/feature or product used in this lab - no need to repeat what you covered in the introduction. Keep this section fairly concise. If you find yourself needing more than to sections/paragraphs, please utilize the "Learn More" section.

### Objectives

*List objectives for this lab using the format below*

In this lab, you will:
* Create graph metadata tables that contain information about the application tables, and which we will be able to use to create a graph and perform community detection in a later lab
* Compute the affinities between the tables

### Prerequisites (Optional)

This lab assumes you have:
* An Oracle account
* All previous labs successfully completed


## Task 1: Create Graph Metadata Tables

In this task, we will create a set of metadata tables that we will use to store the information we need to perform community detection in the next lab.  We will create one table called `TABLESET_TABLES` that will contain a list of all of the tables that were used in the workload capture and how many times each table was accessed and/or participated in a join.  A second table called `TABLE_MAP` will be used to store the affinities between pairs of tables.  Later, when we create a graph, the first table will describe the vertices and the second table will describe the edges.

1. Create the graph metadata tables by running the following statements - make sure you run these in your `TKDRADATA` SQL Worksheet (not the `ADMIN` user's worksheet):

    ```
    <copy>drop table tableset_tables;
    drop table table_map;

    create table tableset_tables 
    ( table_set_name       varchar2(128)
    , schema               varchar2(128)
    , table_name           varchar2(128)
    , total_sql            number(10)
    , total_executions     number(10)
    , tables_joined        number(10));

    create table table_map 
    ( table_set_name       varchar2(128)
    , table1               varchar2(128)
    , schema1              varchar2(128)
    , table2               varchar2(128)
    , schema2              varchar2(128)
    , join_count           number(10)
    , join_executions      number(10)
    , static_coefficient   decimal(10,5)
    , dynamic_coefficient  decimal(10,5)
    , total_affinity       decimal(10,5));</copy>
    ```

2. In this step, we will populate the `TABLESET_TABLES` table, which will become the vertices in our graph.  Execute the following commands to populate the table:

    ```
    <copy>truncate table tableset_tables;

    insert into tableset_tables (table_set_name, schema, table_name, total_sql, total_executions) 
    select table_set_name, table_owner, table_name, count(distinct sql_id), sum(executions)
    from ( 
        select distinct table_set_name, table_owner, table_name, sql_id, executions 
        from (
            select 'tkdradata' table_set_name,
                case when v.operation='INDEX' then v.TABLE_NAME
                     when v.operation='TABLE ACCESS' then v.object_name
                     else NULL end table_name,
                v.object_owner as table_owner,
                v.sql_id,
                v.executions
            from (
                select p.object_name, p.operation, p.object_owner, 
                    p.sql_id, p.executions, i.table_name
                from dba_sqlset_plans p, all_indexes i
                where p.object_name=i.index_name(+) 
                and sqlset_name='tkdradata'
                and object_owner = upper('tkdradata')
            ) v  
        )
    ) 
    group by table_set_name, table_owner, table_name
    having table_name is not null;</copy>
    ```

4. We will also create a helper view that we will use in the affinity calculation.  Execute the following command to create the view:

    ```
    create view tableset_sql as 
    select distinct table_name, sql_id 
    from (
        select 'tkdradata' table_set_name,
        case when v.operation='INDEX' then v.TABLE_NAME
            when v.operation='TABLE ACCESS' then v.object_name
            else NULL end table_name,
        v.object_owner as table_owner,
        v.sql_id,
        v.executions
        from ( 
            select p.object_name, p.operation, p.object_owner,
                p.sql_id, p.executions, i.table_name
            from dba_sqlset_plans p, all_indexes i
            where p.object_name=i.index_name(+) 
            and sqlset_name='tkdradata' 
            and object_owner = 'TKDRADATA'
        ) v
    );
    ```

## Task 2: Compute Affinities

1. Create the procedure to compute the affinities between tables.  This is the procedure we will use to compute the affinities.  This procedure reads the information from the SQL Tuning Set that we created in the previous lab and calculates the affinity between tables based on how many times they are used in SQL statements together, and how they are used.

    The procedure does the following:

    * Get a list of the SQL statements that were run in the captured workload and which used at least one of the tables that we are interested in, either by reading an index or the table itself,
    * For each pair of tables, work out how many times that pair was used in a join and how many times they were used overall,
    * Work out what fraction of statements this pair was used in,
    * Apply weights for joins and executions (50% each) and calculate the total affinity between the tables,
    * For each table, work out how many other tables it was joined to in total.

    TODO - add some info here on what the static/dynamic coeffients mean

    Execute the following statements to create the procedure:

    ```
    create or replace procedure compute_affinity_tkdra as
    cursor c is
    select table_name, schema from tableset_tables;
    tblnm varchar2(128);
    ins_sql varchar2(2000);
    upd_sql varchar2(2000);
    begin
        for r in c loop
            ins_sql:= q'{
                insert into tkdradata.table_map 
                ( table_set_name
                , table1
                , schema1
                , table2
                , schema2
                , join_count
                , join_executions
                , static_coefficient
                , dynamic_coefficient
                , total_affinity) 
                select 
                    'tkdradata' table_set_name,
                    tbl1, 
                    'tkdradata', 
                    tbl2, 
                    'tkdradata', 
                    join_count, 
                    join_executions, 
                    round(join_count/(all_sql-join_count),5) static_coefficient, 
                    round(join_executions/(all_executions-join_executions),5) dynamic_coefficient, 
                    (round(join_count/(all_sql-join_count),5)*0.5 + 
                     round(join_executions/(all_executions-join_executions),5)*0.5) total_affinity
                from (
                    select 
                        v2.tbl1, 
                        v2.tbl2, 
                        (select sum(total_sql) 
                            from tableset_tables 
                            where table_name=v2.tbl1 
                            or table_name=v2.tbl2 ) all_sql,
                        (select sum(total_executions) 
                            from tableset_tables 
                            where table_name=v2.tbl1 
                            or table_name=v2.tbl2 ) all_executions,
                        v2.join_count, 
                        v2.join_executions 
                    from (
                        select 
                            v1.tbl1, 
                            v1.tbl2, 
                            count(distinct v1.sql_id) join_count, 
                            sum(v1.executions) join_executions 
                        from (
                            select distinct 
                                v.tbl1, 
                                case when v.operation='INDEX' then v.TABLE_NAME  
                                    when v.operation='TABLE ACCESS' then v.tbl2 
                                    else NULL end tbl2,
                                sql_id,
                                executions 
                            from ( 
                                select '}'||r.table_name||q'{' tbl1, 
                                s.object_name tbl2, 
                                i.table_name table_name, 
                                sql_id, 
                                operation, 
                                executions 
                            from dba_sqlset_plans s, all_indexes i 
                            where sqlset_name='tkdradata' 
                            and object_owner=upper('tkdradata') 
                            and s.object_name = i.index_name(+) 
                            and sql_id in (
                                select distinct sql_id 
                                from dba_sqlset_plans 
                                where sqlset_name='tkdradata' 
                                and object_name='}'||r.table_name||q'{' 
                                and  object_owner=upper('tkdradata')
                            ) 
                        ) v 
                    ) v1  
                    group by v1.tbl1, v1.tbl2   
                    having v1.tbl2 is not null 
                    and v1.tbl1 <> v1.tbl2 ) v2 
                )
            }';
            execute immediate ins_sql;

            upd_sql:= q'{
                update tkdradata.tableset_tables 
                set tables_joined=(select count(distinct table_name) 
                from (
                    select 
                        'tkdradata' table_set_name,
                        case when v.operation='INDEX' then v.TABLE_NAME 
                            when v.operation='TABLE ACCESS' then v.object_name 
                            else NULL end table_name,
                        v.object_owner as table_owner,
                        v.sql_id, 
                        v.executions 
                    from ( 
                        select 
                            p.object_name, 
                            p.operation, 
                            p.object_owner, 
                            p.sql_id, 
                            p.executions, 
                            i.table_name 
                        from dba_sqlset_plans p, all_indexes i 
                        where p.object_name=i.index_name(+) 
                        and sqlset_name='tkdradata' 
                        and sql_id in (
                            select sql_id 
                            from tableset_sql 
                            where table_name='}'||r.table_name||q'{') 
                            and object_owner = upper('tkdradata')
                        ) v
                    )
                ) where table_name ='}' || r.table_name || q'{'
            }';
            execute immediate upd_sql;
        end loop;

    end;
    /
    ```

2. Run the procedure to compute affinities.

    ```
    <copy>exec compute_affinity_tkdra();</copy>
    ```

    This may take a few minutes to complete.  Once it is done, we can see the output in our two graph metadata tables, for example:

    ```
    <copy>select * from table_map where table1 = 'DRA_1';</copy>
    ```

Once this has been completed you are ready to **proceed to the next lab.**


## Learn More

*(optional - include links to docs, white papers, blogs, etc)*

* [URL text 1](http://docs.oracle.com)
* [URL text 2](http://docs.oracle.com)

## Acknowledgements
- **Author** - Mayank Tayal, Developer Advocate
- **Contributors** - Mark Nelson, Developer Evangelist
- **Last Updated By/Date** - Mayank Tayal, April 2022
