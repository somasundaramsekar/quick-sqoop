#Getting Started
**Download and Install Sqoop**

> **Note**: Choose appropriate version, Do not use sqoop2 as it is not
> officially GA and may never be

    ~$ wget http://apache.arvixe.com/sqoop/1.4.6/sqoop-1.4.6.bin__
    hadoop-2.0.4-alpha.tar.gz
    ~$ sudo mv sqoop-1.4.6.bin__hadoop-2.0.4-alpha.tar.gz /srv/
    ~$ cd /srv
    /srv$ sudo tar -xvf sqoop-1.4.6.bin__hadoop-2.0.4-alpha.tar.gz
    /srv$ sudo chown -R hadoop:hadoop sqoop-1.4.6.bin__hadoop-2.0.4-alpha
    /srv$ sudo ln -s $(pwd)/sqoop-1.4.6.bin__hadoop-2.0.4-alpha $(pwd)/sqoop

> **NOTE**: If you feel setting up your own cluster is overwhelming, download and use [Cloudera Quickstart
> VM](https://www.cloudera.com/downloads/quickstart_vms/5-8.html) and skip the setup part, Quickstart vm setup is as simple as download the archive file for virtual box and choosing **File --> Import Appliance --> [Choose the extracted vmdk file]** 
> 1. Quickstart VM is preloaded with all the necessary Hadoop ecosystem
> services (HDFS, Yarn, Hive, Impala, Beeline, HBase, Spark etc.)
> 2. Default login credentials are  Username: cloudera Password: cloudera, for Mysql username: root, password: cloudera

Use user hadoop and configure environment

    /srv$ sudo su hadoop
    
    $ vim ~/.bashrc
    
    # add Sqoop aliases in bashrc
    export SQOOP_HOME=/srv/sqoop
    export PATH=$PATH:$SQOOP_HOME/bin
   
    $ source ~/.bashrc


> **NOTE**: Place appropriate JDBC driver jar in $SQOOP_HOME/lib directory

verify installation 

    /srv$ cd $SQOOP_HOME
    /srv/sqoop$ sqoop help
    
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

**Simple import**

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

**Protecting the password**
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

**Import as binary**

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
**Compressing**


     #With default gz compression
     $ sqoop import --connect jdbc:mysql://localhost/sqoop --username sqoop --target-dir /etl/input/cities --table cities --compress -P
        
     #With bz2 compression using a different codec
     $ sqoop import --connect jdbc:mysql://localhost/sqoop --username sqoop --target-dir /etl/input/cities --table cities --compress -P 
    --compression-codec org.apache.hadoop.io.compress.BZip2Codec

> **Note**: if in the mapred-site.xml file, the property mapred.output.compress is set to false with the final flag, then Sqoop won’t be able to compress the output files even when you call it with the --compress parameter

**Using --direct to speed up transfer**

    sqoop import \
      --connect jdbc:mysql://localhost/sqoop \
      --username sqoop \
      --table cities \
      --direct
--direct uses native utilities, if provided my the DB vendor to perform the transfer, ex. mysqldump

**Type mapping**
Sqoop lets you map type between the DB and HDFS, this ensures any type incompatibility that may arise are handled at the data ingestion itself

    sqoop import \
      --connect jdbc:mysql://localhost/sqoop \
      --username sqoop \
      --table cities \
      --map-column-java id=Long,country=String,city=String \
      --target-dir /etl/input/cities \
      --compress \
      -P
**Controlling Parallelism**
Sqoop lets you control parallelism by increasing the number of mappers that will read from DB, that is concurrent tasks that will be executed against the DB, it may not always have desired effect, as it can increase the load and may adversely affect the performance, however if sqoop can find that there are no enough data to run given parallel task, it will fall back to the default number

    sqoop import \
      --connect jdbc:mysql://localhost/sqoop \
      --username sqoop \
      --table cities \
      --target-dir /etl/input/cities \
      --num-mappers 10 \
      -P
**Importing null**
by default sqoop imports null as string 'null', but that may differ between environments and your target format and storage, to handle sqoop provides --null-string and --null-non-string options, by default HDFS represents nulls as \N so the null has to be mapped to \N when importing to HDFS

    sqoop import \
      --connect jdbc:mysql://localhost/sqoop \
      --username sqoop \
      --table cities \
      --target-dir /etl/input/cities \
      --null-string '\\N' \
      --null-non-string '\\N' \
      -P

**Import all tables**
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

**Incremental Import**
It is always necessary and pertinent that, there should always be a way to incrementally import changes rather that importing the whole database everytime we import data into HDFS, importing only the deltas will save a load on both DB and HDFS 

sqoop provides `--incremental` option for this purpose, and it comes with two modes
1. append
2. lastmodified

**append**: append is for immutable tables, that just adds new record and does not modifies the existing records, ex: audit tables, log tables etc.

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

**lastmodified**: is used for mutable tables, it imports all rows whose timestamp column specified with `--check-column`, date or time values if greater than the value specified with the `--last-value` option
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

**Sqoop Jobs**
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

** Configuring metastore**
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
**Import with Free form queries**
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


