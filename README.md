#Getting Started
**Download and Install Sqoop**

> **Note**: Choose appropriate version, Do not use sqoop2 as it is not
> officially GA and may never be

    $ wget http://apache.arvixe.com/sqoop/1.4.6/sqoop-1.4.6.bin__hadoop-2.0.4-alpha.tar.gz
    $ sudo mv sqoop-1.4.6.bin__hadoop-2.0.4-alpha.tar.gz /srv/
    $ cd /srv
    $ sudo tar -xvf sqoop-1.4.6.bin__hadoop-2.0.4-alpha.tar.gz
    $ sudo chown -R hadoop:hadoop sqoop-1.4.6.bin__hadoop-2.0.4-alpha
    $ sudo ln -s $(pwd)/sqoop-1.4.6.bin__hadoop-2.0.4-alpha $(pwd)/sqoop

> **NOTE**: If you feel setting up your own cluster is overwhelming, download and use [Cloudera Quickstart
> VM](https://www.cloudera.com/downloads/quickstart_vms/5-8.html) and skip the setup part, Quickstart vm setup is as simple as download the archive file for virtual box and choosing **File --> Import Appliance --> [Choose the extracted vmdk file]** 
> 1. Quickstart VM is preloaded with all the necessary Hadoop ecosystem
> services (HDFS, Yarn, Hive, Impala, Beeline, HBase, Spark etc.)
> 2. Default login credentials are  Username: cloudera Password: cloudera, for Mysql username: root, password: cloudera

Use user hadoop and configure environment

    $ sudo su hadoop
    
    $ vim ~/.bashrc
    
    # add Sqoop aliases in bashrc
    $ export SQOOP_HOME=/srv/sqoop
    $ export PATH=$PATH:$SQOOP_HOME/bin
   
    $ source ~/.bashrc


> **NOTE**: Place appropriate JDBC driver jar in $SQOOP_HOME/lib directory

verify installation 

    $ cd $SQOOP_HOME
    $ sqoop help
    
    15/06/04 21:57:40 INFO sqoop.Sqoop: Running Sqoop version: 1.4.6
    usage: sqoop COMMAND [ARGS]
    
    Available commands:
      codegen            Generate code to interact with database records
      create-hive-table  Import a table definition into Hive
      eval               Evaluate a SQL statement and display the results
      export             Export an HDFS directory to a database table
      help               List available commands
      import             Import a table from a database to HDFS
      import-all-tables  Import tables from a database to HDFS
      job                Work with saved jobs
      list-databases     List available databases on a server
      list-tables        List available tables in a database
      merge              Merge results of incremental imports
      metastore          Run a standalone Sqoop metastore
      version            Display version information
    
    See 'sqoop help COMMAND' for information on a specific command.

If you see any warnings displayed pertaining to HCatalog, you can safely ignore them for now. As you can see, Sqoop provides a list of import- and export-specific commands and tools that expect to connect with either a database or Hadoop data source.

#Importing with Sqoop

> **NOTE**: Use the command  hadoop fs -rmr /etl/input/* to clear the HDFS data, for a clean run on the hands-on excercises.

> **NOTE**: Import excercise seed data and users by login to mysql with 
> '$ mysql -u root -p' password is cloudera, after login execute mysql> source /path/to/file/mysql.credentials.sql and mysql> source /path/to/file/mysql.tables.sql


##Simple import
Sqoop provides import tools to import from your DB to HDFS, Sqoop by default uses JDBC for connecting to the target DB, hence any DB with a JDBC driver can be used with sqoop.

    sqoop import \
      --connect jdbc:mysql://mysql.example.com/sqoop \
      --username sqoop \
      --password sqoop \
      --table cities 
    
Sqoop offers two parameters for specifying custom output directories: --target-dir and --warehouse-dir. Use the --target-dir parameter to specify the directory on HDFS where Sqoop should import your data

    sqoop import \
      --connect jdbc:mysql://localhost/sqoop \
      --username sqoop \
      --password sqoop \
      --table cities \
      --target-dir /etl/input/cities
    
    sqoop import \
      --connect jdbc:mysql://localhost/sqoop \
      --username sqoop \
      --password sqoop \
      --table cities \
      --warehouse-dir /etl/input/
Use the command-line parameter --where to specify a SQL condition that the imported data should meet.

    sqoop import \
      --connect jdbc:mysql://mysql.example.com/sqoop \
      --username sqoop \
      --password sqoop \
      --table cities \
      --where "country = 'USA'"

> **NOTE**: By default, Sqoop will create a directory with the same name as the imported table inside your home directory on HDFS and import
> all data there. For example, when the user jarcec imports the table
> cities, it will be stored in /user/jarcec/cities. This directory can
> be changed to any arbitrary directory on your HDFS using the
> --target-dir parameter. The only requirement is that this directory must not exist prior to running the Sqoop command.

##Protecting the password

The mysql password as plain text is not recommended approach, there are a couple of alternatives, reading the password from the stdin

    sqoop import \
          --connect jdbc:mysql://localhost/sqoop \
          --username sqoop \
          --table cities \
          -P
but this cannot be used as a part of automation scripts as it requires user intervention to provide the password, other alternative is to use the `--password-file` option

First create a password file, that is read-only for the user who is creating the file in HDFS


    $ echo -n "password" > .password
    $ hadoop fs -put .password /user/$USER/
    $ hadoop fs -chmod 400 /user/$USER/.password
    $ rm .password

 and using the password file in sqoop commands
 

     sqoop import \
          --connect jdbc:mysql://localhost/sqoop \
          --username sqoop \
          --password-file /user/$USER/.password \
          --table cities

##Import as binary

Hadoop supports various binary formats like Sequencefile, Avro, Parquet etc, the hadoop supported binary formats can be efficiently split, allowing you to run MapReduce or Spark jobs on the data efficiently(in parallel) across the cluster.

    sqoop import \
      --connect jdbc:mysql://localhost/sqoop \
      --username sqoop \
      --password sqoop \
      --table cities \
      --as-sequencefile
    
    sqoop import \
      --connect jdbc:mysql://localhost/sqoop \
      --username sqoop \
      --password-file /user/$USER/sqoop.password \
      --table cities \
      --as-avrodatafile \
      --target-dir /etl/input/cities-avro 
	  
##Compress

Compressed files occupy less storage space, but with a caveat, compressed files except LZO, bz2 cannot be split and hence disallow parallel processing on the HDFS data.

     #With default gz compression
     $ sqoop import --connect jdbc:mysql://localhost/sqoop --username sqoop --target-dir /etl/input/cities --table cities --compress -P
        
     #With bz2 compression using a different codec
     $ sqoop import --connect jdbc:mysql://localhost/sqoop --username sqoop --target-dir /etl/input/cities --table cities --compress -P 
    --compression-codec org.apache.hadoop.io.compress.BZip2Codec

> **Note**: if in the mapred-site.xml file, the property mapred.output.compress is set to false with the final flag, then Sqoop won’t be able to compress the output files even when you call it with the --compress parameter

##Using --direct to speed up transfer

    sqoop import \
      --connect jdbc:mysql://localhost/sqoop \
      --username sqoop \
      --table cities \
      --direct
	  
--direct uses native utilities, if provided my the DB vendor to perform the transfer, ex. mysqldump

##Type mapping

Sqoop lets you map type between the DB and HDFS, this ensures any type incompatibility that may arise are handled at the data ingestion itself

    sqoop import \
      --connect jdbc:mysql://localhost/sqoop \
      --username sqoop \
      --table cities \
      --map-column-java id=Long,country=String,city=String \
      --target-dir /etl/input/cities \
      --compress \
      -P
	  

##Controlling Parallelism

Sqoop lets you control parallelism by increasing the number of mappers that will read from DB, that is concurrent tasks that will be executed against the DB, it may not always have desired effect, as it can increase the load and may adversely affect the performance, however if sqoop can find that there are no enough data to run given parallel task, it will fall back to the default number

    sqoop import \
      --connect jdbc:mysql://localhost/sqoop \
      --username sqoop \
      --table cities \
      --target-dir /etl/input/cities \
      --num-mappers 10 \
      -P
	  
##Importing null

by default sqoop imports null as string 'null', but that may differ between environments and your target format and storage, to handle sqoop provides --null-string and --null-non-string options, by default HDFS represents nulls as \N so the null has to be mapped to \N when importing to HDFS

    sqoop import \
      --connect jdbc:mysql://localhost/sqoop \
      --username sqoop \
      --table cities \
      --target-dir /etl/input/cities \
      --null-string '\\N' \
      --null-non-string '\\N' \
      -P

##Import all tables

Sqoop allows import of entire database with a single command

    sqoop import-all-tables \
      --connect jdbc:mysql://localhost/sqoop \
      --username sqoop \
      --exclude-tables visits \
      --warehouse-dir /etl/input \
      --num-mappers 10 \
      --null-string '\\N' \
      --null-non-string '\\N' \
      -P 

> **NOTE**: 
 > 1. use --exclude-tables to exclude a selected tables, specify
    comma-separated list of table names
> 2. --target-dir which is used to specify data dir for a single table can no longer be used, use --warehouse-dir instead to specify target dir for all the tables

##Incremental Import

It is always necessary and pertinent that, there should always be a way to incrementally import changes rather that importing the whole database everytime we import data into HDFS, importing only the deltas will save a load on both DB and HDFS 

sqoop provides `--incremental` option for this purpose, and it comes with two modes
1. append
2. lastmodified

**append**

append is for immutable tables, that just adds new record and does not modifies the existing records, ex: audit tables, log tables etc.

    sqoop import \
      --connect jdbc:mysql://localhost/sqoop \
      --username sqoop \
      --table visits \
      --warehouse-dir /etl/input \
      --password-file /user/$USER/.password \
      --incremental append \
      --check-column id \
      --last-value 1
the above command will import all the rows after the specified `--last-value` of 1 from the `--check-column` id

**lastmodified**

Is used for mutable tables, it imports all rows whose timestamp column specified with `--check-column`, date or time values if greater than the value specified with the `--last-value` option
The command runs two jobs, one to import all the matching rows into a temp location and second to merge the values imported with the current values

    sqoop import \
      --connect jdbc:mysql://localhost/sqoop \
      --username sqoop \
      --table visits \
      --warehouse-dir /etl/input \
      --password-file /user/$USER/.password \
      --incremental lastmodified \
      --check-column last_update_date \
      --last-value "2013-05-22 01:01:01" \
      --merge-key id
  
`--merge-key` specify the key to used for merging the modifications 

##Sqoop Jobs

Sqoop jobs lets you automate the incremental imports by allowing sqoop to preserve the state of the job and reexecute from the last saved state, thus in this case saving the `--last-value` across executions, thus reducing the manual overhead.

    sqoop job \
      --create visits \
      -- \
     import \
      --connect jdbc:mysql://localhost/sqoop \
      --username sqoop \
      --table visits \
      --warehouse-dir /etl/input \
      --password-file /user/$USER/.password \
      --incremental append \
      --check-column id \
      --last-value 0 \
      --merge-key id
List the jobs that are stored in the sqoop metastore

    sqoop job --list
Describe a sqoop job

    sqoop job --show visits
execute a job

    sqoop job --exec visits

##Configuring metastore

You can configure metastore to use a persistent DB like mysql in sqoop-site.xml

    <configuration>
            ...
      <property>
        <name>sqoop.metastore.client.autoconnect.url</name>
        <value>jdbc:hsqldb:hsql://your-metastore:16000/sqoop</value>
      </property>
    </configuration>
and the sqoop clients running across machines can use this metastore by

    sqoop job
      --create visits \
      --meta-connect jdbc:mysql//localhost/sqoop_meta \
      -- \
      import \
      --table visits
      ...
	  
##Import with Free form queries

It is possible to import with free form queries, specially useful if you have a normalized set of tables and you want to import materialized (denormalized) view of the table into Hadoop, you replace the `--table` with `--query`.

> **Note**: try avoiding complex queries however as they unnecessary overhead to your ETL, materialize the query into a temp table and import from the temp table.

    sqoop import \
      --connect jdbc:mysql://localhost/sqoop \
      --username sqoop \
      --password-file /user/$USER/.password \
      --query 'SELECT normcities.id, countries.country, normcities.city FROM normcities JOIN countries USING(country_id) WHERE $CONDITIONS' \
      --split-by id \
      --target-dir /etl/input/cities \
      --mapreduce-job-name import-normalized-cities \
      --boundary-query "select min(id), max(id) from normcities"

To achieve parallelism sqoop by default runs `min()`, `max()` on the primary-key to slice the data and run concurrent mappers, with free-form queries, the min max has to be run by materializing the query, which has significant cost, to avoid, you can specify query to identify the boundary values with `--boundary-query`, and you are free to select from anywhere and any table, the first column is used as a min and the second max.

> **NOTE**: `--split-by` is used specify the column to be used for slicing the data for parallel tasks

The `$CONDITIONS` given with the where clause in `--query` will be used by the mapper tasks to specify the slicing conditions for concurrent mappers.

> **NOTE**: To avoid ambiguity in column names across tables while using free-form queries, assign alias to the column names

#Exporting with Sqoop
Sqoop also let's you export data from HDFS onto your DB. To do so the table must already exist in your DB and it need not be empty, but the data you are offloading shouldn't violate any constraint

    sqoop export \
      -Dsqoop.export.records.per.statement=10 \
      -Dsqoop.export.statements.per.transaction=10 \
      --connect jdbc:mysql://localhost/sqoop_export \
      --username sqoop \
      --password-file /user/$USER/.password \
      --table cities \
      --export-dir /etl/input/cities \
      --staging-table staging_cities \
      --batch \
      --clear-staging-table
sqoop uses `export` tool to export the data, you specify the `--table` to export and the HDFS directory from which it is supposed to export using `--export-dir` , by default the export happens row-by-row which has a lot of overhead interms of connections and transactions, to workaround that use `--batch` which batches multiple rows into a single statement, you could also control the number of rows sent per query using `-Dsqoop.export.records.per.statement=10` and number of rows to be inserted before the transaction is committed using `-Dsqoop.export.statements.per.transaction=10`, these 3 parameters lets you optimize the inserts based on the underlying DB.

`--staging-table` lets you specify a staging table, which should be as same as the target table interms of columns and column definitions, the records are inserted into staging table first, once all the mapreduce jobs are successfully completed, the rows are then transferred to the target table. `--clear-staging-table` ensures the staging table is truncated (if supported by the DB) before the export.

##Updates

If the data in the HDFS is mutable, sqoop provides mechanism to update the existing data

    sqoop export \
    	-Dsqoop.export.records.per.statement=10 \
    	-Dsqoop.export.statements.per.transaction=10 \
    	--connect jdbc:mysql://localhost/sqoop_export \
    	--username sqoop \
    	--password-file /user/$USER/.password \
    	--table cities \
    	--export-dir /etl/input/cities \
    	--update-key id \
    	--update-mode allowinsert \
    	--batch
`--update-key` can be used to specify a comma separated list of columns to be used as the look-up key for the row selection. Ex. if a table has columns c1, c2, c3, c4 and `--update-key c3,c4` is given the resulting `UPDATE` would be 

    UPDATE table SET c1 = ?, c2 = ? WHERE c3 = ? and c4 = ?

The lookup values, obviously need not be altered.

If there is a case where there are new columns are added to the source HDFS file, it will not be exported as a part of the UPDATE(of-course!!!), to handle this use `--update-mode allowinsert` , to use this feature however it must be supported by the target DB, right now Oracle and Mysql (without `--direct` mode). The update cannot be used arbitrary updates, however, only solution is to truncate the target table and do a full export.

**Exporting only a subset of columns**

If there is a mismatch between the HDFS data and the target table we can use `--columns` to specify the comma separated list of columns to export, given the target DB supports specifying list of columns during INSERT.

    sqoop export \
    	-Dsqoop.export.records.per.statement=10 \
    	-Dsqoop.export.statements.per.transaction=10 \
    	--connect jdbc:mysql://localhost/sqoop_export \
    	--username sqoop \
    	--password-file /user/$USER/.password \
    	--table cities \
    	--export-dir /etl/input/cities \
    	--columns id,country,city \
    	--update-key id \
    	--update-mode allowinsert \
		--input-null-string '\\N' \
	    --input-null-non-string '\\N' \
    	--batch 
additionally the target table must either allow or have a default value configured. 

Like `import`, `export` also allows you to specify the null character encoding using `--input-null-string` for string based columns and `--input-null-non-string` for non-string based columns.

**Using Stored Procedures**

    sqoop export \
      --connect jdbc:mysql://localhost/sqoop \
      --username sqoop \
      --password sqoop \
      --call populate_cities
      
you can use stored procedures to export instead of `--table` the SP will be called with the number of columns returned from HDFS data, in that order. Be aware of the overhead that SP may cause if it is resource intensive as SP will be called multiple times by mappers running in parallel. 

