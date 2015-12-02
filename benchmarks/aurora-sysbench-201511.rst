.. _aurora-sysbench-2015:

================================
Amazon Aurora Sysbench benchmark
================================

Benchmark date: Nov 2015.

The goal was to evaluate Amazon Aurora performance comparing to Percona Server.
The workload is `sysbench <https://github.com/akopytov/sysbench>`_ v0.5.
Scripts used for testing:

* oltp.lua (read-write)
* update_index.lua
* update_non_index.lua

Percona Server version: 5.6.27-75.0

Data sizes
-----------

Initial dataset: 32 sysbench tables, 50mln rows each. It corresponds to about 400GB of data.

Testing sizes: for benchmarks we vary max amount of rows used by sysbench: 1mln, 2.5mln, 5mln, 10mln, 25mln, 50mln.

In the chart results marked in thousands of rows as: 1000, 2500, 5000, 10000, 25000, 50000.
That is `1000` corresponds to 1mln of rows.

Estimated datasizes:

+--------------------+------+------+------+-------+-------+-------+
| rows, in thousand: | 1000 | 2500 | 5000 | 10000 | 25000 | 50000 |
+--------------------+------+------+------+-------+-------+-------+
| Size in GB:        |    8 |   20 |   40 |    80 |   200 |   400 |
+--------------------+------+------+------+-------+-------+-------+

In this way we emulate different datasizes from fully in-memory (1mln rows) to heavy-IO access (50 mln rows)

Instance sizes.
---------------
Actually it is quite complicated to find equal configuration (in both performance and price aspects)
to compare Percona Server running on EC2 instance against Amazon Aurora.

Amazon Aurora:

* db.r3.xlarge instance (4 virtual CPUS + 30GB memory)
* Monthly computing Cost (1-YEAR TERM, No Upfront): $277.40
* Monthly Storage cost: $0.100 per GB-month * 400 GB = $40
* extra $0.200 per 1 million IO requests

Total cost (per month, excluding extra per IO requests): `$311.40`


Percona Server
r3.xlarge instance (4 virtual CPUS + 30GB memory)
Monthly computing cost (1-YEAR TERM, No Upfront): $160.6

For the storage we will use 3 options:

* general purpose SSD volume (marked as `ps` in charts), 500GB size, 1500/3000 ios, cost: $0.10 per GB-month * 500 = $50
* Provisioned IOPS SSD volume (marked as `ps-io3000`), 500GB, 3000 IOP = $0.125 per GB-month  * 500 + $0.065 per provisioned IOPS-month * 3000 = $62.5 + $195 = $257.5
* Provisioned IOPS SSD volume (marked as `ps-io2000`), 500GB, 2000 IOP = $0.125 per GB-month  * 500 + $0.065 per provisioned IOPS-month * 2000 = $62.5 + $130 = $192.5

So corresponding total costs (per month) for used EC2 instances are: `$210.6`; `$418.1`; `$353.1`

Client setup
------------

For client we use t2.medium instance.
Sysbench runs with 16 users threads, which should be adequate to load the database instance.


sysbench script:

.. code-block:: bash

	for i in 50000 25000 10000 5000 2500 1000
	do
	sysbench --test=tests/db/oltp.lua --oltp_tables_count=32 --mysql-user=root --mysql_table_engine=InnoDB --num-threads=16 --oltp-table-size=${i}000 --rand-type=pareto --rand-init=on --report-interval=10 --mysql-host=HOST --mysql-db=sbtest --max-time=7200 --max-requests=0 run | tee -a au.${i}.oltp.txt
	done

Results
-------

Results for Amazon Aurora; 2 hours run, 10 sec resolution, to show variance of results

.. image:: aurora-sysbench-201511/Aurora-timeline.png
	:width: 800px
	:height: 1200px

Results for Percona Server; 2 hours run, 10 sec resolution, to show variance of results

.. image:: aurora-sysbench-201511/PerconaServer-timeline.png
	:width: 800px
	:height: 1200px


Results (averaged) Percona Server vs Amazon Aurora, in relation to datasize

.. image:: aurora-sysbench-201511/PerconaServer-vs-Aurora.png
	:width: 800px
	:height: 800px

Observations
------------

There are few points to highlight:

* Even in long runs (2 hours) I do not see a fluctuation in results; the throughput is stable
* I actually made one run for 48 hour, still no fluctuations
* For Percona Server, as expected, better storage gives better throughput, especially for IO-heavy cases
* Amazon Aurora shows worse results with smaller datasizes; and Aurora outperforms Percona Server (with general purpose SSD and provisioned SSD 2000IOPS volumes)
in big datasizes.
* It looks like Amazon Aurora does not benefit from adding extra memory - the throughput does not grow much with small datasizes.
I think it proves my assumption that Aurora has some kind of write-through cache, which shows better results in IO-heavy workloads.

Appendix
--------

Percona Server my.cnf file:

	[mysqld]

	table-open-cache-instances=32
	table_open_cache=8000


	innodb-flush-method            = O_DIRECT
	innodb-log-files-in-group      = 2
	innodb-log-file-size           = 16G
	innodb-flush-log-at-trx-commit = 1
	innodb_log_compressed_pages     =0

	innodb-file-per-table          = 1
	innodb-buffer-pool-size        = 20G

	innodb_write_io_threads        = 8
	innodb_read_io_threads         = 32
	innodb_open_files              = 1024

	innodb_old_blocks_pct           =10
	innodb_old_blocks_time          =2000

	innodb_checksum_algorithm = crc32

	innodb_file_format              =Barracuda

	innodb_io_capacity=1500
	innodb_io_capacity_max=2000
	metadata_locks_hash_instances=256
	innodb_max_dirty_pages_pct=90
	innodb_flush_neighbors=1
	innodb_buffer_pool_instances=8
	innodb_lru_scan_depth=4096
	innodb_sync_spin_loops=30
	innodb-purge-threads=16
