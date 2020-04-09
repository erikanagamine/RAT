
# Real Application Testing

Real Application Testing, aka RAT, is an option introduced on oracle 11g.

This option comprehend the follow features:

* Database Replay
* SQL Performance analyzer (SPA)

In this document, we consider:
* orcl11gr2-demo as a source server
* ora12cr1-demo as a target server

In this document we will execute the how to use the features:

__Database Replay__

For this demonstration you will need at least 2 databases (one for source and one for target) and a space for files.

In the source database, you will need to configure the capture process.

___Configure capture process___

1. Creating directory:
<!-- markdownlint-disable MD01 -->
[oracle@orcl11gr2-demo ~]$ cd /u01/app/oracle

[oracle@orcl11gr2-demo oracle]$ mkdir dbcapture

[oracle@orcl11gr2-demo oracle]$ cd dbcapture/
<!-- markdownlint-enable MD01 -->

1. Creating directory on database

Logging in the database, using the client of your preference (in this case I will use the SQL*Plus):
<!-- markdownlint-disable MD02 -->
SQL> create directory dbcapture as '/u01/app/oracle/dbcapture/';
Directory created.
<!-- markdownlint-enable MD02 -->

3. Start capture process

Before start the capture process we recommend to shutdown the database and take a backup if you doing this for a production database, you can use others techiques together (like a physical backup and data guard).

For this workshop we will only start the capture process. To this process we will use the DBMS_WORKLOAD_CAPTURE package with admintrator user:

<!-- markdownlint-disable MD03 -->
BEGIN
  DBMS_WORKLOAD_CAPTURE.start_capture (
      name => 'test_capture_1',
       dir => 'DBCAPTURE',
  duration => NULL);
  END;
/
<!-- markdownlint-disable MD04 -->

4. Start work to capture

Now we will start some work to capture. First we will create a user:

``CREATE USER db_replay_test IDENTIFIED BY Welcome123
  QUOTA UNLIMITED ON users;
  
GRANT CONNECT, CREATE TABLE TO db_replay_test;
``

Now we will connect with this user and generate a load:

``SQL> conn db_replay_test/Welcome123
  SQL> CREATE TABLE db_replay_test_tab (
   id           NUMBER,
  description  VARCHAR2(50),
  CONSTRAINT db_replay_test_tab_pk PRIMARY KEY (id)
);

SQL> BEGIN
  FOR i IN 1 .. 500000 LOOP
    INSERT INTO db_replay_test_tab (id, description)
    VALUES (i, 'Description for ' || i);
  END LOOP;
  COMMIT;
END;
/``

5. Finish the capture process

After the load, connect as a administrator and stop the capture:

``BEGIN
   DBMS_WORKLOAD_CAPTURE.finish_capture;
   END;
   /
``

6. Transfer the capture generated to target server

``[oracle@orcl11gr2-demo dbcapture]$ ls -l
total 8
drwxr-xr-x 2 oracle asmadmin 4096 Mar 30 04:54 cap
drwxr-xr-x 3 oracle asmadmin 4096 Mar 30 04:08 capfiles
[oracle@orcl11gr2-demo dbcapture]$ cd ..
[oracle@orcl11gr2-demo oracle]$ scp -r dbcapture  10.0.0.4:/u01/app/oracle
wcr_scapture.wmd                                                                                                                                           100%  102    90.1KB/s   00:00
wcr_cr.text                                                                                                                                                100%   19KB  10.9MB/s   00:00
wcr_cr.xml                                                                                                                                                 100%  188KB  43.1MB/s   00:00
wcr_cr.html                                                                                                                                                100%   47KB  21.3MB/s   00:00
wcr_fcapture.wmd                                                                                                                                           100%  208   196.9KB/s   00:00
wcr_82w6mh0000001.rec                                                                                                                                      100%   41KB  19.2MB/s   00:00
wcr_82wpgh000000a.rec                                                                                                                                      100% 3040     2.7MB/s   00:00
wcr_82yu7h000000r.rec                                                                                                                                      100% 1651     1.3MB/s   00:00
wcr_82wswh000000b.rec                                                                                                                                      100% 4103     3.3MB/s   00:00
``

You can checking the capture process on database.

``SQL> COLUMN name FORMAT A30
SELECT id, name FROM dba_workload_captures;SQL>

'        ID NAME
' ---------- ------------------------------
         1 test_capture_1

SQL>  SELECT DBMS_WORKLOAD_CAPTURE.get_capture_info('DBCAPTURE') from dual;

DBMS_WORKLOAD_CAPTURE.GET_CAPTURE_INFO('DBCAPTURE')
 '---------------------------------------------------
                                                  1
                                                  
``

Also, you can generate report:

``DECLARE
  l_report  CLOB;
BEGIN
  l_report := DBMS_WORKLOAD_CAPTURE.report(capture_id => 1,
                                           format     => DBMS_WORKLOAD_CAPTURE.TYPE_HTML);
END;
/``


And export this report to HTML:

``
BEGIN
  DBMS_WORKLOAD_CAPTURE.export_awr (capture_id => 1);
END;
/``



___Configure replay process___

1. Checking generated files and files having been transfer to target

Checking all files that have been generated during capture process and transfered to the target server:

``
[oracle@ora12cr1-demo ~]$ cd /u01/app/oracle
[oracle@ora12cr1-demo oracle]$ cd dbcapture/
[oracle@ora12cr1-demo dbcapture]$ ls -l
total 8
drwxr-xr-x 2 oracle oinstall 4096 Mar 31 18:13 cap
drwxr-xr-x 3 oracle oinstall 4096 Mar 30 04:57 capfiles
``

2. Create directory on target database

You need to create a database link to import data generated on Database Replay:

``SQL> create directory dbcapture as '/u01/app/oracle/dbcapture/';
Directory created.``

3. Preparing the target database to Replay

With this procedure, you will indicate to the target database with all generated logs and appoint where the logs resides.

``BEGIN
  DBMS_WORKLOAD_REPLAY.process_capture('DBCAPTURE');

  DBMS_WORKLOAD_REPLAY.initialize_replay (replay_name => 'test_capture_1',
                                          replay_dir  => 'DBCAPTURE');

  DBMS_WORKLOAD_REPLAY.prepare_replay (synchronization => TRUE);
END;
/
``

4. Checking if the target database has sufficient resources to replay the workload

Before to start the replay process, it's important to

``[oracle@orcl11gr2-demo dbcapture]$ wrc mode=calibrate replaydir=/u01/app/oracle/dbcapture

Workload Replay Client: Release 11.2.0.4.0 - Production on Wed Apr 1 00:57:05 2020

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.


Report for Workload in: /u01/app/oracle/dbcapture
-----------------------

Recommendation:
Consider using at least 1 clients divided among 1 CPU(s)
You will need at least 7 MB of memory per client process.
If your machine(s) cannot match that number, consider using more clients.

Workload Characteristics:
- max concurrency: 2 sessions
- total number of sessions: 4

Assumptions:
- 1 client process per 50 concurrent sessions
- 4 client process per CPU
- 256 KB of memory cache per concurrent session
- think time scale = 100
- connect time scale = 100
- synchronization = TRUE
 ``
 
5. Replay a workload: instantiate a client

Once checked the workload, you have to instantiate the workload replay (observe the amount of CPU necessary to workload):

``
[oracle@ol6-12cr1 ~]$ wrc system/oracle mode=replay replaydir=/u01/app/oracle/dbcapture

Workload Replay Client: Release 12.1.0.2.0 - Production on Wed Apr 8 17:23:23 2020

Copyright (c) 1982, 2014, Oracle and/or its affiliates.  All rights reserved.


Wait for the replay to start (17:23:23)

``
Leave this session opened to monitoring the replay process once started. Also, you can use the nohup to this process.


6. Replay a workload: Start

Once client instantiation, you need to start the replay:

``
[oracle@ol6-12cr1 ~]$ sqlplus

SQL*Plus: Release 12.1.0.2.0 Production on Wed Apr 8 17:27:31 2020

Copyright (c) 1982, 2014, Oracle.  All rights reserved.

Enter user-name: system/oracle
Last Successful login time: Wed Apr 08 2020 17:23:23 +00:00

Connected to:
Oracle Database 12c Enterprise Edition Release 12.1.0.2.0 - 64bit Production
With the Partitioning, OLAP, Advanced Analytics and Real Application Testing options

SQL> BEGIN
  DBMS_WORKLOAD_REPLAY.start_replay;
END;
/  2    3    4

PL/SQL procedure successfully completed.

SQL>
``
PS. you can cancel this process once started with DBMS_WORKLOAD_REPLAY.cancel_replay;

After this command, on the other session when replay finishes, you will find this message:

``[oracle@ol6-12cr1 dbcapture]$ wrc system/oracle mode=replay replaydir=/u01/app/oracle/dbcapture/

Workload Replay Client: Release 12.1.0.2.0 - Production on Wed Apr 8 19:45:00 2020

Copyright (c) 1982, 2014, Oracle and/or its affiliates.  All rights reserved.


Wait for the replay to start (19:45:00)
Replay client 1 started (19:45:58)
Replay client 1 finished (21:30:00)``



__Additional  materials__
 Database Capture and Replay: Common Errors and Reasons (Doc ID 463263.1)

 Mandatory Patches for Database Testing Functionality for Current and Earlier Releases (Doc ID 560977.1)
 
DBMS_REPLAY - https://docs.oracle.com/en/database/oracle/oracle-database/18/arpls/DBMS_WORKLOAD_REPLAY.html#GUID-FE03A123-2257-41FF-BA90-AD0114DC1A4F

How to Use Database Replay Feature to Help With The Upgrade From 10.2.0.4 to 11g (Doc ID 748895.1)
