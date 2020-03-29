# Database Replay in Oracle Database 11g Release 1

The Database Replay functionality of Oracle 11g allows you to capture workloads on a production system and replay them exactly as they happened on a test system. This provides an accurate method to test the impact of a variety of system changes including:

-   Database upgrades.
-   Operating system upgrades or migrations.
-   Configuration changes, such as changes to initialization parameters or conversion from a single node to a RAC environment.
-   Hardware changes or migrations.

The capture and replay processes can be configured and initiated using PL/SQL APIs, or Enterprise Manager, both of which are demonstrated in this article. To keep things simple, the examples presented here are performed against two servers (prod-11g and test-11g), both of which run an identical database with a SID of DB11G.

-   [Capture using the DBMS_WORKLOAD_CAPTURE Package](https://oracle-base.com/articles/11g/database-replay-11gr1#capture_using_api)
-   [Replay using the DBMS_WORKLOAD_REPLAY Package](https://oracle-base.com/articles/11g/database-replay-11gr1#replay_using_api)
-   [Capture using Enterprise Manager](https://oracle-base.com/articles/11g/database-replay-11gr1#capture_using_em)
-   [Replay using Enterprise Manager](https://oracle-base.com/articles/11g/database-replay-11gr1#replay_using_em)

## Capture using the DBMS_WORKLOAD_CAPTURE Package

The  `DBMS_WORKLOAD_CAPTURE`  package provides a set of procedures and functions to control the capture process. Before we can initiate the capture process we need an empty directory on the "prod-11g" database server to hold the capture logs.

mkdir /u01/app/oracle/db_replay_capture

Next, we create a directory object pointing to the new directory.

CONN sys/password@prod AS SYSDBA

CREATE OR REPLACE DIRECTORY db_replay_capture_dir AS '/u01/app/oracle/db_replay_capture/';

-- Make sure existing processes are complete.
SHUTDOWN IMMEDIATE
STARTUP

Notice the inclusion of a shutdown and startup of the database. This is not necessary, but Oracle suggest it as a good way to make sure any outstanding processes are complete before starting the capture process.

The combination of the  `ADD_FILTER`  procedure and the  `DEFAULT_ACTION`  parameter of the  `START_CAPTURE`  procedure allow the workload to be refined by including or excluding specific work based on the following attributes:

-   INSTANCE_NUMBER
-   USER
-   MODULE
-   ACTION
-   PROGRAM
-   SERVICE

For simplicity let's assume we want to capture everything, so we can ignore this and jump straight to the  `START_CAPTURE`  procedure. This procedure allows us to name a capture run, specify the directory the capture files should be placed in, and specify the length of time the capture process should run for. If the duration is set to NULL, the capture runs until it is manually turned off using the  `FINISH_CAPTURE`  procedure.

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

Once the work is complete we can stop the capture using the  `FINISH_CAPTURE`  procedure.

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

We can retrieve the  `ID`  of the capture run by passing the directory object name to the  `GET_CAPTURE_INFO`  function, or by querying the  `DBA_WORKLOAD_CAPTURES`  view.

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

The  `DBA_WORKLOAD_CAPTURES`  view contains information about the capture process. This can be queried directly, or a report can be generated in text or HTML format using the  `REPORT`  function.

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

## Replay using the DBMS_WORKLOAD_REPLAY Package

The  `DBMS_WORKLOAD_REPLAY`  package provides a set of procedures and functions to control the replay process. In order to replay the logs captured on the "prod-11g" system, we need to transfers the capture files to our test system. Before we can do this, we need to create a directory on the "test-11g" system to put them in. For simplicity we will keep the name the same.

mkdir /u01/app/oracle/db_replay_capture

Transfer the files from the production server to the test server.

Next, we create a directory object pointing to the new directory.

CONN sys/password@test AS SYSDBA

CREATE OR REPLACE DIRECTORY db_replay_capture_dir AS '/u01/app/oracle/db_replay_capture/';

It is a good idea to adjust the test system time to match the time when the capture process was started. This way any time-based processing will react in the same way. For this test I have ignored this step.

We can now prepare to replay the existing capture logs using the  `PROCESS_CAPTURE`,  `INITIALIZE_REPLAY`  and  `PREPARE_REPLAY`  procedures. I've named the replay with the same name as the capture process (test_capture_1), but this is not necessary.

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

If you need to stop the replay before it is complete, call the  `CANCEL_REPLAY`  procedure.

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

Information about the replay processing is available from the  `DBA_WORKLOAD_REPLAYS`  view.

COLUMN name FORMAT A30
SELECT id, name FROM dba_workload_replays;

        ID NAME
---------- ------------------------------
        11 test_capture_1

1 row selected.

SQL>

In addition, a report can be generated in text or HTML format using the  `REPORT`  function.

DECLARE
  l_report  CLOB;
BEGIN
  l_report := DBMS_WORKLOAD_REPLAY.report(replay_id => 11,
                                          format     => DBMS_WORKLOAD_REPLAY.TYPE_HTML);
END;
/

## Capture using Enterprise Manager

As you might expect, the capture processing using Enterprise Manager is similar to that of using the PL/SQL APIs. First you must create a directory and a directory object on the production server. We are reusing those created for the PL/SQL API example, which is fine provided you delete the contents of the directory before you start a new capture.

Next, log on to Enterprise Manager on the production server (prod-11g) and navigate to "Software and Support" section, then click on the "Database Replay" link.

![Database Replay Link](https://oracle-base.com/articles/11g/images/db_replay/01-DatabaseReplayLink.jpg)

On the "Database Replay" page, click the "Capture Workload" task icon.

![Capture Workload](https://oracle-base.com/articles/11g/images/db_replay/02-CaptureWorkload.jpg)

Check the acknowledge check boxes and click the "Next" button.

![Plan Environment](https://oracle-base.com/articles/11g/images/db_replay/03-PlanEnvironment.jpg)

Accept the default options by clicking the "Next" button.

![Options](https://oracle-base.com/articles/11g/images/db_replay/04-Options.jpg)

Enter a capture name and select the correct directory object, then click the "Next" button.

![Parameters](https://oracle-base.com/articles/11g/images/db_replay/05-Parameters.jpg)

Enter the host credentials on the "Schedule" page, then click the "Next" button.

![Schedule](https://oracle-base.com/articles/11g/images/db_replay/06-Schedule.jpg)

Review the capture information, then click the "Submit" button.

![Review](https://oracle-base.com/articles/11g/images/db_replay/07-Review.jpg)

Wait while the capture process is initiated.

![Processing](https://oracle-base.com/articles/11g/images/db_replay/08-Processing.jpg)

The "Database Replay" screen reappears with a confirmation message informing you that the capture process has been initiated.

![Database Replay](https://oracle-base.com/articles/11g/images/db_replay/09-DatabaseReplay.jpg)

Click the refresh button intermittently until the "Active Capture and Replay" section of the screen lists the capture process. Once this is present you can do some processing to be captured. In this example I did a repeat of the work done in the PL/SQL API capture section.

CONN sys/password@prod AS SYSDBA

DROP USER db_replay_test CASCADE;

CREATE USER db_replay_test IDENTIFIED BY db_replay_test
  QUOTA UNLIMITED ON users;
  
GRANT CONNECT, CREATE TABLE TO db_replay_test;

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

When it is time to stop the capture process, return to the "Database Replay" screen and click on the "Stop" button in the "Active Capture and Replay" section.

![Active Capture](https://oracle-base.com/articles/11g/images/db_replay/10-ActiveCapture.jpg)

Wait while the capture process stops, then click "Yes" to export the AWR data.

![Export AWR Data](https://oracle-base.com/articles/11g/images/db_replay/11-ExportAWRData.jpg)

Once the AWR export job has been submitted, you are directed to the "View Workload Capture" screen, which gives a summary of the capture process.

![View Workload Capture](https://oracle-base.com/articles/11g/images/db_replay/12-ViewWorkloadCapture.jpg)

Clicking on "View Workload Capture Report" button will display a report similar to  [this](https://oracle-base.com/articles/11g/images/db_replay/workload_report_capture.html).

The AWR export is not complete at this point, it is only scheduled. To monitor the job click on the "Database" tab, move to the "Server" subsection and click on the "Jobs" link.

![Scheduler Jobs](https://oracle-base.com/articles/11g/images/db_replay/13-SchedulerJobs.jpg)

On "Scheduler Jobs" screen, click on the "Running" link to view running jobs only. Click the "Refresh" button intermittently until the AWR export job is no longer listed as running.

![Running Job](https://oracle-base.com/articles/11g/images/db_replay/14-RunningJob.jpg)

The capture process is now complete.

## Replay using Enterprise Manager

As with the PL/SQL API replay example, we must create a directory and directory object on the test server to hold the capture files from the production server. If you are using an existing directory, as we are, you must make sure it is empty before transferring the capture files to it.

Once the capture files are in place, log on to Enterprise Manager and navigate to the Database Replay screen. Click on the "Preprocess Captured Workload" task icon.

![Preprocess Captured Workload](https://oracle-base.com/articles/11g/images/db_replay/15-PreprocessCapturedWorkload.jpg)

Select the directory object that points to the capture files. The screen will immediately update with information about the capture files. Click the "Preprocess Workload" button.

![Directory](https://oracle-base.com/articles/11g/images/db_replay/16-Directory.jpg)

Click the "Next" button on the "Database Version" screen.

![Database Version](https://oracle-base.com/articles/11g/images/db_replay/17-DatabaseVersion.jpg)

Enter the host credentials in the "Schedule" screen, then click the "Next" button.

![Schedule](https://oracle-base.com/articles/11g/images/db_replay/18-Schedule.jpg)

Assuming everything looks OK, click the "Submit" button.

![Review](https://oracle-base.com/articles/11g/images/db_replay/19-Review.jpg)

Eventually, you return to the "Database Replay" screen with a confirmation that the preprocess job has been scheduled. This job executes almost immediately, so you will probably not have to check the running jobs screen to see if it is still running.

![Database Replay](https://oracle-base.com/articles/11g/images/db_replay/20-DatabaseReplay.jpg)

On the "Database Replay" screen, click the "Replay Workload" task icon.

![Replay Workload](https://oracle-base.com/articles/11g/images/db_replay/21-ReplayWorkload.jpg)

Select the appropriate directory object. The screen will update immediately to reflect the contents of the selected directory. Click the "Set up Replay" button.

![Directory](https://oracle-base.com/articles/11g/images/db_replay/22-Directory.jpg)

Read the prerequisites, then click the "Continue" button.

![Prerequisites](https://oracle-base.com/articles/11g/images/db_replay/23-Prerequisites.jpg)

Click the "Continue" button on the "References to External Sytems" screen.

![References To External Sytems](https://oracle-base.com/articles/11g/images/db_replay/24-ReferencesToExternalSytems.jpg)

Enter a replay name, then click the "Next" button. In this example I made the replay name match the capture name, but it is not necessary.

![Initial Options](https://oracle-base.com/articles/11g/images/db_replay/25-InitialOptions.jpg)

Accept the default customized options by clicking the "Next" button.

![Customize Options](https://oracle-base.com/articles/11g/images/db_replay/26-CustomizeOptions.jpg)

Click the "Next" button on the "Prepare Replay Clients" screen.

![Prepare Replay Clients.jpg](https://oracle-base.com/articles/11g/images/db_replay/27-PrepareReplayClients.jpg)

The "Wait For Client Connections" screen continually refreshes looking for Replay Client processes. At this point you must start the replay clients manually on the test server.

$ wrc system/password@test mode=replay replaydir=/u01/app/oracle/db_replay_capture

Workload Replay Client: Release 11.1.0.6.0 - Production on Tue Oct 30 09:34:14 2007

Copyright (c) 1982, 2007, Oracle.  All rights reserved.


Wait for the replay to start (09:34:14)

Once the replay client has started it will be shown in the "Wait For Client Connections" screen and you can click the "Next" button.

![Wait For Client Connections.jpg](https://oracle-base.com/articles/11g/images/db_replay/28-WaitForClientConnections.jpg)

Click the "Submit" button on the "Review" screen. Notice the recommendation to alter the system time to match the capture start time. For a real replay this would make sense, as it may affect any time-based processing, but for this example it can be ignored.

![Review](https://oracle-base.com/articles/11g/images/db_replay/29-Review.jpg)

Wait while the replay process is initiated.

![Processing](https://oracle-base.com/articles/11g/images/db_replay/30-Processing.jpg)

The "View Workload Replay" screen will refresh repeatedly until the replay process is complete. Once complete, the page contains a comparison of the processing done during the capture and the replay.

![](https://oracle-base.com/articles/11g/images/db_replay/31-ViewWorkloadReplay.jpg)

Clicking on "View Workload Replay Report" button will display a report similar to  [this](https://oracle-base.com/articles/11g/images/db_replay/workload_report_replay.html).

The comparison between the capture and replay processing allows you to determine if a system change has affected performance of the system.

For more information see:

-   [Database Replay](http://docs.oracle.com/cd/E11882_01/server.112/e41481/rat_intro.htm)
-   [DBMS_WORKLOAD_CAPTURE](http://docs.oracle.com/cd/E11882_01/appdev.112/e40758/d_workload_capture.htm)
-   [DBMS_WORKLOAD_REPLAY](http://docs.oracle.com/cd/E11882_01/appdev.112/e40758/d_workload_replay.htm)


> Written with [StackEdit](https://stackedit.io/).
