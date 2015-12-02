.. _aurora-sysbench-2015:

================================
Amazon Aurora Sysbench benchmark
================================

Benchmark date: Nov 2015.

The goal was to evaluate Amazon Aurora performance compared to Percona Server.
The workload is `sysbench <https://github.com/akopytov/sysbench>`_ v0.5.
Scripts used for testing:

* oltp.lua (read-write)
* update_index.lua
* update_non_index.lua

Percona Server version: 5.6.27-75.0

Data Sizes
-----------

* **Initial dataset**. 32 sysbench tables, 50 million (mln) rows each. It corresponds to about 400GB of data.

* **Testing sizes**. For benchmarks, we vary the maximum amount of rows used by sysbench: 1mln, 2.5mln, 5mln, 10mln, 25mln, 50mln.

In the chart, the results are marked in thousands of rows: 1000, 2500, 5000, 10000, 25000, 50000.
In other words, "1000" corresponds to 1mln rows.

Estimated datasizes:

+--------------------+------+------+------+-------+-------+-------+
| rows, in thousand: | 1000 | 2500 | 5000 | 10000 | 25000 | 50000 |
+--------------------+------+------+------+-------+-------+-------+
| Size in GB:        |    8 |   20 |   40 |    80 |   200 |   400 |
+--------------------+------+------+------+-------+-------+-------+

This is how we were able to emulate different datasizes, from fully in-memory (1mln rows) to heavy-IO access (50 mln rows).

Instance Sizes.
---------------
It is actually very complicated to find an equal configuration (in both performance and price aspects)
to use as a comparison between Percona Server running on an EC2 instance, and Amazon Aurora.

Amazon Aurora:

* db.r3.xlarge instance (4 virtual CPUS + 30GB memory)
* Monthly computing cost (1-YEAR TERM, No Upfront): $277.40
* Monthly storage cost: $0.100 per GB-month * 400 GB = $40
* extra $0.200 per 1 million IO requests

Total cost (per month, excluding extra per IO requests): $311.40


Percona Server
r3.xlarge instance (4 virtual CPUS + 30GB memory)
Monthly computing cost (1-YEAR TERM, No Upfront): $160.60

For the storage we will use 3 options:

* General purpose SSD volume (marked as "ps" in charts), 500GB size, 1500/3000 ios, cost: $0.10 per GB-month * 500 = $50
* Provisioned IOPS SSD volume (marked as "ps-io3000"), 500GB, 3000 IOP = $0.125 per GB-month  * 500 + $0.065 per provisioned IOPS-month * 3000 = $62.5 + $195 = $257.5
* Provisioned IOPS SSD volume (marked as "ps-io2000"), 500GB, 2000 IOP = $0.125 per GB-month  * 500 + $0.065 per provisioned IOPS-month * 2000 = $62.5 + $130 = $192.5

So corresponding total costs (per month) for used EC2 instances are: $210.60; $418.10; $353.10

Client Setup
------------

For the client we use t2.medium instance.
Sysbench runs with 16 users threads, which should be adequate to load the database instance.


sysbench script:

.. code-block:: bash

	for i in 50000 25000 10000 5000 2500 1000
	do
	sysbench --test=tests/db/oltp.lua --oltp_tables_count=32 --mysql-user=root --mysql_table_engine=InnoDB --num-threads=16 --oltp-table-size=${i}000 --rand-type=pareto --rand-init=on --report-interval=10 --mysql-host=HOST --mysql-db=sbtest --max-time=7200 --max-requests=0 run | tee -a au.${i}.oltp.txt
	done

Results
-------

Results for Amazon Aurora: 2 hours run, 10 second resolution, to show variance of results.

.. image:: aurora-sysbench-201511/Aurora-timeline.png
	:width: 800px
	:height: 1200px

Results for Percona Server: 2 hours run, 10 second resolution, to show variance of results.

.. image:: aurora-sysbench-201511/PerconaServer-timeline.png
	:width: 800px
	:height: 1200px


Results (averaged) for Percona Server vs. Amazon Aurora, in relation to datasize:

.. image:: aurora-sysbench-201511/PerconaServer-vs-Aurora.png
	:width: 800px
	:height: 800px

Results in tabular format, where I also added an "ops cost" column. This is calculated as "instance_cost_per_month"/"ops" (less is better):


=========  =====  ================  =========  =========
Server      Size  workload               ops    ops cost
=========  =====  ================  =========  =========
aurora      1000  oltp              1548.6490  0.2010785
ps          1000  oltp              2894.6190  0.0727557
ps-io2000   1000  oltp              2903.5302  0.1216106
ps-io3000   1000  oltp              2889.8827  0.1446772
aurora      2500  oltp              1653.0761  0.1883761
ps          2500  oltp              2009.9911  0.1047766
ps-io2000   2500  oltp              2809.9707  0.1256597
ps-io3000   2500  oltp              2783.0859  0.1502289
aurora      5000  oltp              1340.6739  0.2322712
ps          5000  oltp               955.1150  0.2204970
ps-io2000   5000  oltp              1452.4108  0.2431130
ps-io3000   5000  oltp              2132.9517  0.1960194
aurora     10000  oltp              1139.2001  0.2733497
ps         10000  oltp               596.4517  0.3530881
ps-io2000  10000  oltp               912.3408  0.3870264
ps-io3000  10000  oltp              1420.1322  0.2944092
aurora     25000  oltp               919.9039  0.3385136
ps         25000  oltp               418.1550  0.5036410
ps-io2000  25000  oltp               620.7486  0.5688293
ps-io3000  25000  oltp               964.7347  0.4333834
aurora     50000  oltp               824.9817  0.3774629
ps         50000  oltp               340.6678  0.6181976
ps-io2000  50000  oltp               509.2594  0.6933597
ps-io3000  50000  oltp               782.1511  0.5345514
aurora      1000  update_index      1541.7061  0.2019840
ps          1000  update_index      3230.3314  0.0651945
ps-io2000   1000  update_index      4228.8192  0.0834985
ps-io3000   1000  update_index      4279.5530  0.0976971
aurora      2500  update_index      1440.2293  0.2162156
ps          2500  update_index      2062.2836  0.1021198
ps-io2000   2500  update_index      3105.0119  0.1137194
ps-io3000   2500  update_index      3943.8099  0.1060142
aurora      5000  update_index      1366.3545  0.2279057
ps          5000  update_index      1492.3974  0.1411152
ps-io2000   5000  update_index      2315.1324  0.1525183
ps-io3000   5000  update_index      3465.8786  0.1206332
aurora     10000  update_index      1296.2791  0.2402260
ps         10000  update_index      1189.4472  0.1770570
ps-io2000  10000  update_index      1847.5573  0.1911172
ps-io3000  10000  update_index      2789.7427  0.1498705
aurora     25000  update_index      1209.9782  0.2573600
ps         25000  update_index       938.9404  0.2242954
ps-io2000  25000  update_index      1441.1059  0.2450202
ps-io3000  25000  update_index      2209.2510  0.1892497
aurora     50000  update_index      1140.1554  0.2731207
ps         50000  update_index       804.7809  0.2616861
ps-io2000  50000  update_index      1230.8842  0.2868670
ps-io3000  50000  update_index      1881.0907  0.2222647
aurora      1000  update_non_index  2029.6650  0.1534243
ps          1000  update_non_index  3882.0086  0.0542503
ps-io2000   1000  update_non_index  4181.7776  0.0844378
ps-io3000   1000  update_non_index  4483.4039  0.0932550
aurora      2500  update_non_index  2070.1151  0.1504264
ps          2500  update_non_index  3388.3528  0.0621541
ps-io2000   2500  update_non_index  4247.6228  0.0831289
ps-io3000   2500  update_non_index  4379.8502  0.0954599
aurora      5000  update_non_index  2047.9839  0.1520520
ps          5000  update_non_index  2114.0799  0.0996178
ps-io2000   5000  update_non_index  3359.1397  0.1051162
ps-io3000   5000  update_non_index  4045.8934  0.1033394
aurora     10000  update_non_index  1969.5656  0.1581059
ps         10000  update_non_index  1531.5859  0.1375045
ps-io2000  10000  update_non_index  2372.8680  0.1488073
ps-io3000  10000  update_non_index  3719.5061  0.1124074
aurora     25000  update_non_index  1777.8730  0.1751531
ps         25000  update_non_index  1197.7339  0.1758320
ps-io2000  25000  update_non_index  1780.2134  0.1983470
ps-io3000  25000  update_non_index  2755.3000  0.1517439
aurora     50000  update_non_index  1709.5230  0.1821561
ps         50000  update_non_index  1072.4871  0.1963660
ps-io2000  50000  update_non_index  1428.6931  0.2471489
ps-io3000  50000  update_non_index  2259.8432  0.1850128
=========  =====  ================  =========  =========


Observations
------------

There are few important points to highlight:

* Even in long runs (2 hours) I didn't see a fluctuation in results. The throughput is stable.
* I actually made one run for 48 hours. There were still no fluctuations.
* For Percona Server, as expected, better storage gives better throughput. 3000 IOPS is better then Amazon Aurora, especially for IO-heavy cases
* Amazon Aurora shows worse results with smaller datasizes. Aurora outperforms Percona Server (with general purpose SSD and provisioned SSD 2000IOPS volumes) when it comes to big datasizes.
* It appears that Amazon Aurora does not benefit from adding extra memory - the throughput does not grow much with small datasizes. I think it proves my assumption that Aurora has some kind of write-through cache, which shows better results in IO-heavy workloads.

Appendix
--------

`Raw results and scripts <https://github.com/Percona-Lab/benchmark-results/tree/aurora-sysbench-201511>`_

Percona Server my.cnf file:

::

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
