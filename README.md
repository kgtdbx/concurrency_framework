# concurrency_framework
Framework for processing work concurrently via Snowflake JS stored proc 


[Snowflake stored procedures](https://docs.snowflake.com/en/sql-reference/stored-procedures-overview.html) use JavaScript to express complex logic. The following project shows how we can use a single stored procedure to initiate the execution of multiple statements in parallel by using [Snowflake Tasks](https://docs.snowflake.com/en/user-guide/tasks-intro.html). The idea is to batch/chunk the total amount of work into N batches/chunks. Each batch of work will be handled by 1 task. Multiple tasks can execute on different clusters in the same multi-cluster warehouse. Snowflake will automatically scale out when queuing occurs after MAX_CONCURRENCY_LEVEL (see below) is exceeded. Cross process communiction is accomplished by creating messages in a logging table.

The stored procedure used to distribute the workload takes 4 input parameters.
* method name ('PROCESS_REQUEST') 
* number of worker processes
* number of statements per batch
* name of inputput table 

If the individual statements have a small runtime variance, then we should split the total amount of statements by the number of worker processes. This has the benefit of incurring the startup overhead only once. In contrast, if there is a high variance in execution time per statement, and the number of statements is considerably higher then the number of worker nodes, then it is benefical to send the work to the worker processes in smaller batches. This way, worker nodes do not sit idle because statements may already have been assigned to an already busy worker node. Optimizing the parameters, e.g. testing different parameter configuration can yield major reductions in run-time.

The input table provides the SQL statements to be executed. It has an ID (statement_id) and a JSON document in a variant column (sqlquery). Both names are hard coded and any input table to the framework has to have those 2 attributes. 

In some instances certain statements need to be executed sequentially (like in the sample below). We first execute a "show" which is immediately followed by a result_scan. To support this type of pattern, you define the sql statements in an array.

The example below shows for instance how to get a list of statements to script all roles in an account into a table. The full code can be found in the [samples directory](samples/scripting_roles.sql).

```
select 
seq4() statement_id
,parse_json ('{"Role":"'||"name"||'"'
                     ||',"sqlquery":['
                     ||'"SHOW GRANTS TO ROLE '||"name"||'"'
                     ||',"INSERT INTO concurrency_demo.roles.roles_snapshot SELECT * FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))"'
                     ||']'
                     ||'}') task
from  table(result_scan(last_query_id()));
```

## Set up
To run the sample code in your environment perform the following steps. It is assumed that the user running the set up and executes the code has the necessary permissions. This could be ACCOUNTADMIN or a custom role that has the following permissions.
    ```
    CREATE WAREHOUSE
    CREATE DATABASE
    EXECUTE TASK
    ```
1. Create a warehouse. The concurrency level is reduced to 2 to ensure that the individual cluster isn't overloaded before Snowflake scales out. The Warehouse_size is set to XSMALL. Your specific use-case may require a bigger warehouse size and or more clusters. Modifying MAX_CONCURRENCY_LEVEL may also be beneficial. As a rule of thumb, start small and test different parameters for your workload.   
    ```
    create warehouse concurrency_demo with
       WAREHOUSE_SIZE = XSMALL
       MAX_CLUSTER_COUNT = 10
       SCALING_POLICY = STANDARD
       AUTO_SUSPEND = 15
       AUTO_RESUME = TRUE
       MAX_CONCURRENCY_LEVEL=2;
    ```
1. Clone this repo (use the command below or any other way to clone the repo)
    ```
    git clone https://github.com/RobertFehrmann/concurrency_framework.git
    ```
1. Create a database and install the stored procedure 
    ```
    create database concurrency_demo;
    create schema concurrency_demo.meta_schema
    ```
1. Create procedure sp_concurrent from the meta_schema directory inside the cloned repo by loading the file into a new worksheet and then clicking `Run`. Note: If you are getting an error message (SQL compilation error: parse ...), move the cursor to the end of the file, click into the window, and then click `Run` again). Be sure to set your context correctly, either from the drop downs in your worksheet or by running the the commands below.
    ```
    use database concurrency_demo;
    use warehouse concurrency_demo;
    use role <ACCOUNTADMIN/your own custom role>
    ```

## Testing

For testing purposes, i.e. to see the impact of executing a set of statements concurrently, we will simulate a high IO intensive workload by creating multiple tables with 100 million rows each. 

1. Open a new worksheet and set your context, either from the drop downs in your worksheet or by running the commands below.  
    ```
    use database concurrency_demo;
    use warehouse concurrency_demo;
    use role <ACCOUNTADMIN/your own custom role>
    ```
1. Run test for 1 worker thread, 10 tables, 100 million rows per table. This statement should run for about 12 minutes.
    ```
    create or replace table meta_schema.test1 as 
      select 
      seq4() statement_id
      ,parse_json ('{"TableDB":"CONCURRENCY_DEMO","TableSchema":"TABLE_SCHEMA","Table":"TABLE_'||lpad(statement_id,4,'0')||'"'
                     ||',"rowcount":100000000,"sqlquery":['
                     ||'"CREATE OR REPLACE /* '||lpad(statement_id,4,'0')||' */ '
                     ||' TABLE CONCURRENCY_DEMO.TABLE_SCHEMA.TABLE_'||lpad(statement_id,4,'0')||' AS'
                     ||' SELECT randstr(16,random(11000)+'||statement_id||')::varchar(128) s1 '
                     ||'  ,randstr(16,random(12000)+'||statement_id||')::varchar(128) s2 '
                     ||'  ,randstr(16,random(13000)+'||statement_id||')::varchar(128) s3 '
                     ||' FROM TABLE(generator(rowcount=>100000000))"'
                     ||']'
                     ||'}') task
      from  table(generator(rowcount=>10));
    call meta_schema.sp_concurrent('PROCESS_REQUEST',1,10,'TEST1');
    ```
1. Check table meta_schema.log for the execution log after the call below completes. 
    ```
    select * from meta_schema.log order by id desc;
    ```
1. Run test for 2 worker threads, 10 tables, 100 million rows per table. This statement should run for about 6 minutes, i.e. double the resources for the same amount of work should yield half the run time. 
    ```
    call meta_schema.sp_concurrent('PROCESS_REQUEST',2,5,'TEST1');
    ```
    You can observe progress of the different worker process via the following query
    ```
    select worker_id,worker_session_id, batch_id,status,datediff(s,create_ts,current_timestamp) run_time
    from concurrency_demo.tmp_schema.scheduler 
    where worker_id is not null
    qualify 1=(row_number() over (partition by worker_id order by create_ts desc) )
    order by batch_id ;
    ```
1. Run test for 5 worker thread, 50 tables (!), 100 million rows per table. This statement should run for about 12 minutes, i.e. 5 times the resouces for 5 times the work should yield about the same run time as the first test. This test creates about 1/4 a TB of data and consumes about 1 credit.
    ```
    create or replace table meta_schema.test3 as 
      select 
      seq4() statement_id
      ,parse_json ('{"TableDB":"CONCURRENCY_DEMO","TableSchema":"TABLE_SCHEMA","Table":"TABLE_'||lpad(statement_id,4,'0')||'"'
                     ||',"rowcount":100000000,"sqlquery":['
                     ||'"CREATE OR REPLACE /* '||lpad(statement_id,4,'0')||' */ '
                     ||' TABLE CONCURRENCY_DEMO.TABLE_SCHEMA.TABLE_'||lpad(statement_id,4,'0')||' AS'
                     ||' SELECT randstr(16,random(11000)+'||statement_id||')::varchar(128) s1 '
                     ||'  ,randstr(16,random(12000)+'||statement_id||')::varchar(128) s2 '
                     ||'  ,randstr(16,random(13000)+'||statement_id||')::varchar(128) s3 '
                     ||' FROM TABLE(generator(rowcount=>100000000))"'
                     ||']'
                     ||'}') task
      from  table(generator(rowcount=>50));    
    call meta_schema.sp_concurrent('PROCESS_REQUEST',5,10,'TEST3);
    ```
1. Run test for 10 worker thread, 100 tables (!), 100 million rows per table. This statement should run for about 12 minutes, i.e. 10 times the resources for 10 times the work should yield about the same run time as the first test. This test creates about 1/2 a TB of data and consumes about 2 credits.
    ```
    create or replace table meta_schema.test4 as 
      select 
      seq4() statement_id
      ,parse_json ('{"TableDB":"CONCURRENCY_DEMO","TableSchema":"TABLE_SCHEMA","Table":"TABLE_'||lpad(statement_id,4,'0')||'"'
                     ||',"rowcount":100000000,"sqlquery":['
                     ||'"CREATE OR REPLACE /* '||lpad(statement_id,4,'0')||' */ '
                     ||' TABLE CONCURRENCY_DEMO.TABLE_SCHEMA.TABLE_'||lpad(statement_id,4,'0')||' AS'
                     ||' SELECT randstr(16,random(11000)+'||statement_id||')::varchar(128) s1 '
                     ||'  ,randstr(16,random(12000)+'||statement_id||')::varchar(128) s2 '
                     ||'  ,randstr(16,random(13000)+'||statement_id||')::varchar(128) s3 '
                     ||' FROM TABLE(generator(rowcount=>100000000))"'
                     ||']'
                     ||'}') task
      from  table(generator(rowcount=>100));
    call meta_schema.sp_concurrent('PROCESS_REQUEST',10,10,'TEST4');
    ```

