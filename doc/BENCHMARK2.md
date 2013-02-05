VoltDB Blog: 877,000 TPS with Erlang and VoltDB 
===============================================

Henning Diedrich - 4 Feb 2013

**Running on a suitable EC2 configuration (see details below), with our new VoltDB Erlang driver we achieved 877,519 transactions per second.**

I am Henning Diedrich, CEO of [Eonblast Corporation](http://www.eonblast.com) a games company. I would like to introduce the new Erlang VoltDB driver we created, a piece of software that allows two genre-defining technologies to work together: VoltDB and Erlang.

The Driver
--------------

I first came to VoltDB on the hunt for a better database for heavy duty online-game servers. I experienced first hand what a pain it was to try to scale MySQL and found **VoltDB** uniquely suitable for the requirements of more complex game worlds. Better than any other database in fact.

I had also looked for a better language than Java for programming servers and for that,  **Erlang** had caught my attention. To be able to use them together, I started creating the Erlang driver for VoltDB.

Work for this started three years ago and I donated a first version of the driver to VoltDB at their request in 2010. It was perfectly usable but out of the box only provided for synchronous connections. In 2012 VoltDB decided to sponsor the creation of a bigger and badder version. Now the real deal has arrived.

The benchmark described below was made with the new, asynchronous driver. It is pure Erlang, full of parallel microprocesses, blazingly fast and fit for VoltDB 3. It incorporates almost all of the previous, robust driver version. And on my quest to ensure reliable, consistently high throughput, I was able to draw from my experience maintaining an Erlang **MySQL** driver, Emysql. The connection pooling and call queueing is modeled after the ones used in this reliable workhorse, which was originally designed at Electronic Arts. They enable the Erlang driver to absorb access peaks, and to distribute load across VoltDB server nodes.

To come up with a useful benchmark script I could employ the lessons learned from the Node.js benchmark I did for VoltDB a while ago, see http://blog.voltdb.com/695k-tps-nodejs-and-voltdb/. This time around I knew which numbers I would be looking for and what double checks I should have in place to put the cloud cluster to good use.

The internal structure of the driver has been implemented as would be expected: your program's microprocesses use the functions the driver exposes to send a message to a dedicated connection process, which handles the socket work. After the request is sent, the initiating process is either blocked in a *synchronous* receive (this, of course, does *not* block all your *other* processes) or goes on to to use its time as it pleases, should you choose the *asynchronous* mode. The answer from the server arrives in your processes' mailbox. 

There are many options that you can use. E.g. the 'monitored' mode, where a worker process is created that handles the sending of the request, thereby shielding your initiating process from any hiccups in the driver. You can *fire and forget*, for writes where you don't care to hear that they succeeded. Or *blowout* if you don't even care to hear about failure.

The Benchmark Application
-------------------------------------

The benchmark is based on the VoltDB voter example, which comes with every VoltDB distribution. It 'validates and stores phoned-in votes for talent show contestants'. In the original example setup, there is a web page that displays the results for illustration, updated every 400ms. You'll find it in the `examples/voter` directory of your *VoltDB* installation.

The benchmark starts out with a preparational phase, where the database is filled with 6 contestants' names and then one million write transactions are fired towards the server, per CPU core, that each register a 'vote' for one of the contestant, picked at random. In the end, the votes won by each contestants are displayed, using a materialized view and a VoltDB ad-hoc query.

The benchmark source is under etc/bench of the *driver* home directory, where you'll also find a detailed README.md that explains the multiple was to run the benchmark and make it fit your setup. For a (slow) test run on localhost, it's basically:

    $ git clone git://github.com/VoltDB/voltdb.git voltdb
    $ git clone git://github.com/VoltDB/voltdb-client-erlang.git erlvolt
    $ cd voltdb/examples/voter && ./run.sh &
    $ cd && cd erlvolt && make bench

That should give you a screen like this:

		metal:~ hd$ cd voltdb/examples/voter && ./run.sh &
		[1] 62084
		metal:~ hd$ Initializing VoltDB...
		
		_    __      ____  ____  ____ 
		| |  / /___  / / /_/ __ \/ __ )
		| | / / __ \/ / __/ / / / __  |
		| |/ / /_/ / / /_/ /_/ / /_/ / 
		|___/\____/_/\__/_____/_____/
		
		--------------------------------
		
		WARN: Running 3.0 preview (iv2 mode). NOT SUPPORTED IN PRODUCTION.
		Build: 3.0 voltdb-3.0-beta2-110-g178a1e6 Community Edition
		Connecting to VoltDB cluster as the leader...
		Appointing HSId 0:0 as leader for partition 0
		Appointing HSId 0:1 as leader for partition 1
		MP 0:6 for partition 16383 finished leader promotion. Took 104 ms.
		WARN: Mailbox is not registered for site id -4
		Initializing initiator ID: 0, SiteID: 0:13
		WARN: Running without redundancy (k=0) is not recommended for production use.
		Server completed initialization.

		cd && cd ErlVolt2 && make bench
		erlc -W -I ../include  +native -smp  -o ../ebin erlvolt.erl
		erlc -W -I ../include  +native -smp  -o ../ebin erlvolt_app.erl
		erlc -W -I ../include  +native -smp  -o ../ebin erlvolt_conn.erl
		erlc -W -I ../include  +native -smp  -o ../ebin erlvolt_conn_mgr.erl
		erlc -W -I ../include  +native -smp  -o ../ebin erlvolt_profiler.erl
		erlc -W -I ../include  +native -smp  -o ../ebin erlvolt_sup.erl
		erlc -W -I ../include  +native -smp  -o ../ebin erlvolt_wire.erl
		erlc -W -I ../../include  +native -smp  -o ../../ebin bench.erl
		
		Erlvolt Bench 0.9 (client 'VSD')
		-------------------------------------------------------------------------------------------------------------------------------------
		Client 'VSD', voter, 100,000 calls, steady, 200 workers, delay n/a, direct, queue n/a, slots n/a, limit n/a, verbose, 'n/a' stats/sec
		Hosts: localhost:21212 localhost:21212 
		connect ...
		preparation ...
		calls ... ........................................................................................................................................
		cool down ... 
		check writes ... ok
		results ...  votes:     100,000 (6 contestants)
		....Jessie Alloway:      16,811
		...Tabatha Gehling:      16,661
		....Jessie Eichman:      16,643
		.....Alana Bregman:      16,634
		.....Edwina Burnam:      16,632
		......Kelly Clauss:      16,619
		close pool ...
		Client 'VSD', voter, 100,000 calls, steady, 200 workers, delay n/a, direct, queue n/a, slots n/a, limit n/a, verbose, 'n/a' stats/sec
		-------------------------------------------------------------------------------------------------------------------------------------
		Client 'VSD' overall: 3,657 T/sec throughput, 0.00% fails, total transactions: 100,000, fails: 0, total time: 27.338sec 
		Erlvolt 0.3.0, bench started 2013-01-31 22:09:27, ended 2013-01-31 22:09:54, database: +100,000 new votes
		[+++]
		metal:ErlVolt2 hd$ 


For further instructions, e.g. how to best run the benchmark in the cloud, please see etc/bench/README.me, or verbatim doc/BENCHMARK-README.html, in your driver home folder.


The Benchmark Results
--------------------------------

When run on a single core (-smb +S 1), with a 12-node VoltDB server cluster listening on the other side, the Erlang driver showed throughput of 26,500 transactions per second (TPS) and more. When fully utilizing a 16-core cluster instance as client node, it routinely reached throughput of 260,000 TPS.

Using 8 client nodes connected to a 12-node VoltDB cluster, each client node executed an average of 109,689 transactions per second for a total of 877,519 TPS.

Since this benchmark was about the driver, not the server, I made no attempts to tune the server cluster. After a lot of experimenting, I believe that the lower performance per client core for bigger server clusters, would reflect the network limitations of the EC2 cloud, even for cluster instance where the hope would be that a benchmark would not end up network-bound.

Part of the goal for the benchmark was testing how the driver would hold up under load and that turned out very well. The driver does not crash from really heavy overload and it copes well with 'backpressure' when the server does not allow further requests for being at capacity. However, the fastest benchmarks result when not overloading the server.


The Environment
-----------------------

I started a 20-node Amazon EC2 cc2.xlarge cluster broken up into 8 Erlang client and 12 VoltDB server nodes. The m3.2xlarge provide the following, as described by the Amazon EC2 Instance Types page:

### M3 Double Extra Large Instance

* 30 GiB memory
* 26 EC2 Compute Units (8 virtual cores with 3.25 EC2 Compute Units each)
* 64-bit platform
* I/O Performance: High
* API name: m3.2xlarge

These nodes were configured with:

* Ubuntu Server 12.04 LTS for Cluster Instances AMI
* Oracle JDK 1.7
* Erlang 15
* VoltDB Enterprise Edition 3.0 RC

Each of the five server nodes was set to six partitions, so I had 30 partitions across the cluster.


The Transactions
-----------------------

The clients "firehose" the VoltDB cluster by calling Voter's `vote()` stored procedure continuously. This procedure performes not only one write but, depending on how you count, 4 to 6 operations:

* It retrieves the caller's location (a select)
* Verifies that the caller has not exceeded his/her vote maximum (a select)
* Verifies that the caller was voting for a valid contestant (a select)
* And if yes to all of the above, a vote is cast on behalf of that caller (an insert)

On top of this, each insert also triggers an update to two different materialized views.

Consequently, the 877,000 TPS performed 3.5 million SQL operations per second, i.e. three selects and one insert. 

Observations & Notes
-----------------------------

The most important number from this, to my mind, is the 26,500 transactions per second per CPU core that I saw. This will allow you to make rough estimates on the amount of hardware you may need on the business server side (the VoltDB 'client' side). Your client will usually have more work to do than simply swamp the server, as the benchmark does. So you have an upper limit here and can feel your way down from there.

We decided for the Amazon Elastic Cloud for the benchmark in the hopes that this would result into the most transparent setup. A local cluster of eight "bare metal" nodes would certainly perform better than the EC2 instance, and be way more economic if you used them on a daily basis. But our throughput numbers would be hard to reproduce independently.

As it is, you could try the exact same benchmark yourself, easily. VoltDB and the new driver can be downloaded from http://voltdb.com/community/downloads.php. The README.md of the driver, and its etc/bench/README.md have more instructions on how to use the driver and how to make benchmarks. To find experimental new versions of the driver as well as fast bug fixes, try https://github.com/VoltDB/voltdb-client-erlang. The VoltDB community edition is also on github, at https://github.com/VoltDB/voltdb.

/hd 4 feb 13