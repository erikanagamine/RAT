# RAT
Real Application Testing


The DBMS_WORKLOAD_CAPTURE package provides a set of procedures and functions to control the capture process. Before we can initiate the capture process we need an empty directory on the "prod-11g" database server to hold the capture logs.

mkdir /u01/app/oracle/db_replay_capture
Next, we create a directory object pointing to the new directory.

CONN sys/password@prod AS SYSDBA

CREATE OR REPLACE DIRECTORY db_replay_capture_dir AS '/u01/app/oracle/db_replay_capture/';

-- Make sure existing processes are complete.
SHUTDOWN IMMEDIATE
STARTUP
Notice the inclusion of a shutdown and startup of the database. This is not necessary, but Oracle suggest it as a good way to make sure any outstanding processes are complete before starting the capture process.

The combination of the ADD_FILTER procedure and the DEFAULT_ACTION parameter of the START_CAPTURE procedure allow the workload to be refined by including or excluding specific work based on the following attributes:

INSTANCE_NUMBER
USER
MODULE
ACTION
PROGRAM
SERVICE
For simplicity let's assume we want to capture everything, so we can ignore this and jump straight to the START_CAPTURE procedure. This procedure allows us to name a capture run, specify the directory the capture files should be placed in, and specify the length of time the capture process should run for. If the duration is set to NULL, the capture runs until it is manually turned off using the FINISH_CAPTURE procedure.

BEGIN
  DBMS_WORKLOAD_CAPTURE.start_capture (name     => 'test_capture_1', 
                                       dir      => 'DB_REPLAY_CAPTURE_DIR',
                                       duration => NULL);
END;
/
Now, we need to do some work to capture. First, we create a test user.

CREATE USER db_replay_test IDENTIFIED BY db_replay_test
  QUOTA UNLIMITED ON users;
  
GRANT CONNECT, CREATE TABLE TO db_replay_test;
Next, we create a table and populate it with some data.

CONN db_replay_test/db_replay_test@prod

CREATE TABLE db_replay_test_tab (
  id           NUMBER,
  description  VARCHAR2(50),
  CONSTRAINT db_replay_test_tab_pk PRIMARY KEY (id)
);

BEGIN
  FOR i IN 1 .. 500000 LOOP
    INSERT INTO db_replay_test_tab (id, description)
    VALUES (i, 'Description for ' || i);
  END LOOP;
  COMMIT;
END;
/
Once the work is complete we can stop the capture using the FINISH_CAPTURE procedure.

CONN sys/password@prod AS SYSDBA

BEGIN
  DBMS_WORKLOAD_CAPTURE.finish_capture;
END;
/
If we check out the capture directory, we can see that some files have been generated there.

$ cd /u01/app/oracle/db_replay_capture
$ ls
wcr_4f9rtgw00238y.rec  wcr_cr.html       wcr_scapture.wmd
wcr_4f9rtjw002397.rec  wcr_cr.text
wcr_4f9rtyw00239h.rec  wcr_fcapture.wmd
$
We can retrieve the ID of the capture run by passing the directory object name to the GET_CAPTURE_INFO function, or by querying the DBA_WORKLOAD_CAPTURES view.

SELECT DBMS_WORKLOAD_CAPTURE.get_capture_info('DB_REPLAY_CAPTURE_DIR')
FROM   dual;

DBMS_WORKLOAD_CAPTURE.GET_CAPTURE_INFO('DB_REPLAY_CAPTURE_DIR')
---------------------------------------------------------------
                                                             21

1 row selected.

SQL>

COLUMN name FORMAT A30
SELECT id, name FROM dba_workload_captures;

        ID NAME
---------- ------------------------------
        21 test_capture_1

1 row selected.

SQL>
The DBA_WORKLOAD_CAPTURES view contains information about the capture process. This can be queried directly, or a report can be generated in text or HTML format using the REPORT function.

DECLARE
  l_report  CLOB;
BEGIN
  l_report := DBMS_WORKLOAD_CAPTURE.report(capture_id => 21,
                                           format     => DBMS_WORKLOAD_CAPTURE.TYPE_HTML);
END;
/
The capture ID can be used to export the AWR snapshots associated with the specific capture run.

BEGIN
  DBMS_WORKLOAD_CAPTURE.export_awr (capture_id => 21);
END;
/
A quick look at the capture directory shows a dump file and associated log file have been produced.

$ cd /u01/app/oracle/db_replay_capture
$ ls
wcr_4f9rtgw00238y.rec  wcr_ca.dmp   wcr_cr.text
wcr_4f9rtjw002397.rec  wcr_ca.log   wcr_fcapture.wmd
wcr_4f9rtyw00239h.rec  wcr_cr.html  wcr_scapture.wmd
$
Replay using the DBMS_WORKLOAD_REPLAY Package
The DBMS_WORKLOAD_REPLAY package provides a set of procedures and functions to control the replay process. In order to replay the logs captured on the "prod-11g" system, we need to transfers the capture files to our test system. Before we can do this, we need to create a directory on the "test-11g" system to put them in. For simplicity we will keep the name the same.

mkdir /u01/app/oracle/db_replay_capture
Transfer the files from the production server to the test server.

Next, we create a directory object pointing to the new directory.

CONN sys/password@test AS SYSDBA

CREATE OR REPLACE DIRECTORY db_replay_capture_dir AS '/u01/app/oracle/db_replay_capture/';
It is a good idea to adjust the test system time to match the time when the capture process was started. This way any time-based processing will react in the same way. For this test I have ignored this step.

We can now prepare to replay the existing capture logs using the PROCESS_CAPTURE, INITIALIZE_REPLAY and PREPARE_REPLAY procedures. I've named the replay with the same name as the capture process (test_capture_1), but this is not necessary.

BEGIN
  DBMS_WORKLOAD_REPLAY.process_capture('DB_REPLAY_CAPTURE_DIR');

  DBMS_WORKLOAD_REPLAY.initialize_replay (replay_name => 'test_capture_1',
                                          replay_dir  => 'DB_REPLAY_CAPTURE_DIR');

  DBMS_WORKLOAD_REPLAY.prepare_replay (synchronization => TRUE);
END;
/
Before we can start the replay, we need to calibrate and start a replay client using the "wrc" utility. The calibration step tells us the number of replay clients and hosts necessary to faithfully replay the workload.

$ wrc mode=calibrate replaydir=/u01/app/oracle/db_replay_capture

Workload Replay Client: Release 11.1.0.6.0 - Production on Tue Oct 30 09:33:42 2007

Copyright (c) 1982, 2007, Oracle.  All rights reserved.


Report for Workload in: /u01/app/oracle/db_replay_capture
-----------------------

Recommendation:
Consider using at least 1 clients divided among 1 CPU(s).

Workload Characteristics:
- max concurrency: 1 sessions
- total number of sessions: 3

Assumptions:
- 1 client process per 50 concurrent sessions
- 4 client process per CPU
- think time scale = 100
- connect time scale = 100
- synchronization = TRUE

$
The calibration step suggest a single client on a single CPU is enough, so we only need to start a single replay client, which is shown below.

$ wrc system/password@test mode=replay replaydir=/u01/app/oracle/db_replay_capture

Workload Replay Client: Release 11.1.0.6.0 - Production on Tue Oct 30 09:34:14 2007

Copyright (c) 1982, 2007, Oracle.  All rights reserved.


Wait for the replay to start (09:34:14)
The replay client pauses waiting for replay to start. We initiate replay with the following command.

BEGIN
  DBMS_WORKLOAD_REPLAY.start_replay;
END;
/
If you need to stop the replay before it is complete, call the CANCEL_REPLAY procedure.

The output from the replay client includes the start and finish time of the replay operation.

$ wrc system/password@test mode=replay replaydir=/u01/app/oracle/db_replay_capture

Workload Replay Client: Release 11.1.0.6.0 - Production on Tue Oct 30 09:34:14 2007

Copyright (c) 1982, 2007, Oracle.  All rights reserved.


Wait for the replay to start (09:34:14)
Replay started (09:34:44)
Replay finished (09:39:15)
$
Once complete, we can see the DB_REPLAY_TEST_TAB table has been created and populated in the DB_REPLAY_TEST schema.

SQL> CONN sys/password@test AS SYSDBA
Connected.
SQL> SELECT table_name FROM dba_tables WHERE owner = 'DB_REPLAY_TEST';

TABLE_NAME
------------------------------
DB_REPLAY_TEST_TAB

SQL> SELECT COUNT(*) FROM db_replay_test.db_replay_test_tab;

  COUNT(*)
----------
    500000

SQL>
Information about the replay processing is available from the DBA_WORKLOAD_REPLAYS view.

COLUMN name FORMAT A30
SELECT id, name FROM dba_workload_replays;

        ID NAME
---------- ------------------------------
        11 test_capture_1

1 row selected.

SQL>
In addition, a report can be generated in text or HTML format using the REPORT function.

DECLARE
  l_report  CLOB;
BEGIN
  l_report := DBMS_WORKLOAD_REPLAY.report(replay_id => 11,
                                          format     => DBMS_WORKLOAD_REPLAY.TYPE_HTML);
END;
/
