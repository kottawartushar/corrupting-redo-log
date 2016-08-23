Conn to sys using sqlplus in admin command prompt

C:\WINDOWS\system32>SQLPLUS / AS SYSDBA

Check whether your database is in archive log mode or not. 
Run command ARCHIVE LOG LIST.
It will show you the following details:

SQL> ARCHIVE LOG LIST
Database log mode              Archive Mode
Automatic archival             Enabled
Archive destination            USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     2614
Next log sequence to archive   2619
Current log sequence           2619

It's always preferable to have your database in archive log mode.
The reason being if your database is in archive log mode you can recover from all committed changes in the event of an OS or disk failure.


SQL>COL MEMBER FORMAT A40
SQL>SELECT GROUP#,L.STATUS,V.MEMBER FROM V$LOG L JOIN V$LOGFILE V USING (GROUP#) ORDER BY GROUP#;

This query will display the path of redo log files, their group and their status

    GROUP# STATUS           MEMBER
---------- ---------------- ----------------------------------------
         1 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO04.LOG
         1 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO01.LOG
         2 CURRENT          C:\APP\PC\ORADATA\ADMIN\REDO02.LOG
         2 CURRENT          C:\APP\PC\ORADATA\ADMIN\REDO05.LOG
         3 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO03.LOG
         3 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO08.LOG
         4 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO06.LOG
         4 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO09.LOG
         5 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO10.LOG
         5 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO07.LOG
         6 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO_G6_M1.LOG 


It is always recommended to have a minimum of two members in one group.

As you can see I have only 1 member in group 6 whose current status is INACTIVE.
I intentionally have 1 member to generate a scenario for the sake of this practical.


Now go to the specified path where the redo log member of group 6 is located and open it in a notepad.
It will show some symbolic data. Delete couple of lines and save it the original location which is the one we got as a result of query.


Why did we do the above step i.e. why did we delete some lines of the inactive redo log file?
The reason is we cannot corrupt the redo log file of the member whose status is CURRENT.
Also since I want to explain what is the danger when you have only 1 member in the group and also about the risks of the command which we will be using to restart our database.

Another question is since the status of the member whose redo log file we just corrupted is inactive, then how are we going to check or confirm whether the redo log file is actually corrupt or not and what impact it is going to have on our database?
We certainly cannot wait till that member becomes current to get the error, if any.
So this is what we are going to do: We will force the database to switch the logfile using

SQL>ALTER SYSTEM SWITCH LOGFILE;
SQL>ALTER SYSTEM CHECKPOINT;

Keep repeating above two commands till the member of group 6 becomes CURRENT and other INACTIVE.

When it becomes CURRENT, Run SHUT IMMEDIATE and then START.
You should get the following error/s:


SQL> SHUT IMMEDIATE
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL>
SQL>
SQL> STARTUP
ORACLE instance started.

Total System Global Area 1071333376 bytes
Fixed Size                  1406436 bytes
Variable Size             494930460 bytes
Database Buffers          570425344 bytes
Redo Buffers                4571136 bytes
Database mounted.
ORA-03113: end-of-file on communication channel
Process ID: 5080
Session ID: 5 Serial number: 3


But this is one of the most common errors you will get when learning database.
So how are we going to know that our redo log file is corrupted and that is the actual reason why we cannot startup our database?
I know I know you are all freaked out since the database isn’t starting...
In such case you should recite some famous magical words: 
ABRA KA DABRA!!! 
Keep reciting them until you realize that there is a beautiful text document located in the following location 
C:\app\PC\diag\rdbms\admin\admin\trace (for Windows) 

It is the alert_admin (or alert_<SID> where SID stands for your database name. For example if your database name is orcl then the name will be alert_orcl) text document which records or logs everything that’s happening within your database. 

Open that document in Notepad (for Windows) or VI (for Linux), then scroll down till the end (or press CTRL+END) and look for the following errors:

Errors in file C:\APP\PC\diag\rdbms\admin\admin\trace\admin_lgwr_8104.trc:
ORA-00313: open failed for members of log group 5 of thread 1
ORA-00312: online log 5 thread 1: 'C:\APP\PC\ORADATA\ADMIN\REDO07.LOG'
ORA-27046: file size is not a multiple of logical block size
OSD-04012: file size mismatch (OS 10484100)


And thus we have successfully corrupted our redo log file and the impact of that is we are not able to startup our database!
YIPEEE!! Wait a min I don’t think we should be happy about it.................


Let’s take a look at the steps to resolve these errors:

Since we cannot startup database in current session, open another session and login as sys. We will be connected to an idle instance.
Then start the database in mount mode using:

SQL>STARTUP MOUNT

Then run one of the most dangerous DBA's commands you will ever have to run which is:

SQL>ALTER DATABASE CLEAR UNARCHIVED LOGFILE;

BUT WAIT!!!! Before running the above command let's first understand WHY that particular command?
Isn’t it easier if we just drop that corrupted redo log file? Or why not run this command:

SQL> ALTER DATABASE CLEAR LOGFILE;

The reason is: If the status of your redo log file is CURRENT then there is no archive log file created of that particular redo log file. 
Hence we cannot run ALTER DATABASE CLEAR LOGFILE. 
To check whether your redo logfile is archived or not you can run the following queries:

SQL>SELECT GROUP#,L.STATUS,V.MEMBER,L.SEQUENCE# FROM V$LOG L JOIN V$LOGFILE V USING (GROUP#) ORDER BY GROUP#;

    GROUP# STATUS           MEMBER                                    SEQUENCE#
---------- ---------------- ---------------------------------------- ----------
         1 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO04.LOG             2625
         1 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO01.LOG             2625
         2 CURRENT          C:\APP\PC\ORADATA\ADMIN\REDO02.LOG             2626
         2 CURRENT          C:\APP\PC\ORADATA\ADMIN\REDO05.LOG             2626
         3 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO03.LOG             2621
         3 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO08.LOG             2621
         4 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO06.LOG             2624
         4 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO09.LOG             2624
         5 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO10.LOG             2623
         5 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO07.LOG             2623
         6 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO_G6_M1.LOG         2622


SQL> SELECT GROUP#, ARCHIVED,STATUS FROM V$LOG;

    GROUP# ARC STATUS
---------- --- ----------------
         1 YES INACTIVE
         2 NO  CURRENT
         3 YES INACTIVE
         4 YES INACTIVE
         5 YES INACTIVE
         6 YES INACTIVE

You can see that status of redo log files in group 2 is CURRENT (in 1st query) and a big NO in front of group 2(in 2nd query. Also displays the status, which is CURRENT).

And also database won’t allow you to drop the redo log file whose status is CURRENT. If you try to drop the redo log file whose status is CURRENT it will throw an error. Take a look at the following example:


SQL>SELECT GROUP#,L.STATUS,V.MEMBER,L.SEQUENCE# FROM V$LOG L JOIN V$LOGFILE V USING (GROUP#) ORDER BY GROUP#;

    GROUP# STATUS           MEMBER                                    SEQUENCE#
---------- ---------------- ---------------------------------------- ----------
         1 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO04.LOG             2625
         1 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO01.LOG             2625
         2 CURRENT          C:\APP\PC\ORADATA\ADMIN\REDO02.LOG             2626
         2 CURRENT          C:\APP\PC\ORADATA\ADMIN\REDO05.LOG             2626
         3 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO03.LOG             2621
         3 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO08.LOG             2621
         4 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO06.LOG             2624
         4 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO09.LOG             2624
         5 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO10.LOG             2623
         5 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO07.LOG             2623
         6 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO_G6_M1.LOG         2622


Here status of members of group 2 is CURRENT.
Now if you run the following command:

SQL> ALTER DATABASE DROP LOGFILE MEMBER 'C:\APP\PC\ORADATA\ADMIN\REDO02.LOG';

You will get the result as follows:

ALTER DATABASE DROP LOGFILE MEMBER 'C:\APP\PC\ORADATA\ADMIN\REDO02.LOG'
*
ERROR at line 1:
ORA-01609: log 2 is the current log for thread 1 - cannot drop members
ORA-00312: online log 2 thread 1: 'C:\APP\PC\ORADATA\ADMIN\REDO02.LOG'
ORA-00312: online log 2 thread 1: 'C:\APP\PC\ORADATA\ADMIN\REDO05.LOG'


But then you will say "AHAHAHA I can change the status of group from CURRENT to INACTIVE by forcing a logswitch and then drop the file".
Well BOOO!!! It's not going to help since you will have to drop the entire group. The reason being you cannot just drop the redo log file if it’s the single-lonely member in the group. Take a look at the following example:

SQL> SELECT GROUP#,L.STATUS,V.MEMBER,L.SEQUENCE# FROM V$LOG L JOIN V$LOGFILE V USING (GROUP#) ORDER BY GROUP#;

    GROUP# STATUS           MEMBER                                    SEQUENCE#
---------- ---------------- ---------------------------------------- ----------
         1 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO04.LOG             2625
         1 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO01.LOG             2625
         2 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO02.LOG             2626
         2 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO05.LOG             2626
         3 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO03.LOG             2627
         3 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO08.LOG             2627
         4 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO06.LOG             2624
         4 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO09.LOG             2624
         5 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO10.LOG             2623
         5 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO07.LOG             2623
         6 CURRENT          C:\APP\PC\ORADATA\ADMIN\REDO_G6_M1.LOG         2628


As you can see I have only one member in group 6 whose status is CURRENT. I have already explained and shown why we cannot drop redo log file when its status is CURRENT.
Now I will force a log switch using:

SQL> ALTER SYSTEM SWITCH LOGFILE;

SQL> SELECT GROUP#,L.STATUS,V.MEMBER,L.SEQUENCE# FROM V$LOG L JOIN V$LOGFILE V USING (GROUP#) ORDER BY GROUP#;

    GROUP# STATUS           MEMBER                                    SEQUENCE#
---------- ---------------- ---------------------------------------- ----------
         1 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO04.LOG             2625
         1 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO01.LOG             2625
         2 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO02.LOG             2626
         2 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO05.LOG             2626
         3 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO03.LOG             2627
         3 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO08.LOG             2627
         4 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO06.LOG             2624
         4 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO09.LOG             2624
         5 CURRENT          C:\APP\PC\ORADATA\ADMIN\REDO10.LOG             2629
         5 CURRENT          C:\APP\PC\ORADATA\ADMIN\REDO07.LOG             2629
         6 ACTIVE           C:\APP\PC\ORADATA\ADMIN\REDO_G6_M1.LOG         2628


SQL> ALTER SYSTEM CHECKPOINT;


SQL> SELECT GROUP#,L.STATUS,V.MEMBER,L.SEQUENCE# FROM V$LOG L JOIN V$LOGFILE V USING (GROUP#) ORDER BY GROUP#;

    GROUP# STATUS           MEMBER                                    SEQUENCE#
---------- ---------------- ---------------------------------------- ----------
         1 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO04.LOG             2625
         1 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO01.LOG             2625
         2 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO02.LOG             2626
         2 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO05.LOG             2626
         3 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO03.LOG             2627
         3 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO08.LOG             2627
         4 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO06.LOG             2624
         4 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO09.LOG             2624
         5 CURRENT          C:\APP\PC\ORADATA\ADMIN\REDO10.LOG             2629
         5 CURRENT          C:\APP\PC\ORADATA\ADMIN\REDO07.LOG             2629
         6 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO_G6_M1.LOG         2628


As you can see I have forced the log switch and changed the status from CURRENT to INACTIVE. 
Now look and understand carefully what happens when I try to drop redo log file or member of group 6.

SQL> ALTER DATABASE DROP LOGFILE MEMBER 'C:\APP\PC\ORADATA\ADMIN\REDO_G6_M1.LOG';
ALTER DATABASE DROP LOGFILE MEMBER 'C:\APP\PC\ORADATA\ADMIN\REDO_G6_M1.LOG'
*
ERROR at line 1:
ORA-00361: cannot remove last log member C:\APP\PC\ORADATA\ADMIN\REDO_G6_M1.LOG for group 6

It throws an error saying "cannot remove last log member" meaning you cannot drop redo log member if it is the only member remaining in the group.
Then comes the next question: Can I drop the entire group? Or What happens if I drop the entire group?

The answer to that is it will work but it’s not recommended. The reason being your redo log file of that group is corrupted so you are losing your data anyways. Also if you want the group back you will have recreate the group and add a redo log member thus increasing your own efforts.

So I guess it's time to go back to the most dangerous DBA's command and see what happens after we execute it. Before you execute check your archive log list (You will know the reason later on) and I am going to repeat the steps I mentioned above about corrupting the redo log file and then execute the command. (Geez!! Be patient.) 

Step 1: Check the status of your redo log files.

SQL> SELECT GROUP#,L.STATUS,V.MEMBER FROM V$LOG L JOIN V$LOGFILE V USING (GROUP#) ORDER BY GROUP#;

    GROUP# STATUS           MEMBER
---------- ---------------- ----------------------------------------
         1 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO04.LOG
         1 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO01.LOG
         2 CURRENT          C:\APP\PC\ORADATA\ADMIN\REDO02.LOG
         2 CURRENT          C:\APP\PC\ORADATA\ADMIN\REDO05.LOG
         3 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO03.LOG
         3 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO08.LOG
         4 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO06.LOG
         4 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO09.LOG
         5 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO10.LOG
         5 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO07.LOG
         6 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO_G6_M1.LOG


Step 2: Corrupt the redo log member of the inactive group and for the sake of this practical choose the group which has only member in it.


Step 3: As mentioned earlier you won’t see the corruption error directly on the output screen which is nothing but your command prompt window.
Your command prompt might display the following error:


ORA-01089: immediate shutdown in progress - no operations are permitted
Process ID: 12428
Session ID: 143 Serial number: 1309


You will be able to see the actual error in the alert log document(Please scroll above for the information about alert log document.).

Depending upon how much you have corrupted your redo log file or which lines you have deleted you should get following errors in the alert
log document:

ORA-00327: log 6 of thread 1, physical size 20472 less than needed 20480
ORA-00312: online log 6 thread 1: 'C:\APP\PC\ORADATA\ADMIN\REDO_G6_M1.LOG'


Looking at the error you can make a wild guess that you have deleted something in the redo log file thus causing the reduction in its size.

Step 4: Then perform the following steps:

SQL> SHUT IMMEDIATE

SQL> STARTUP
ORACLE instance started.

Total System Global Area 1071333376 bytes
Fixed Size                  1406436 bytes
Variable Size             713034268 bytes
Database Buffers          352321536 bytes
Redo Buffers                4571136 bytes
Database mounted.
ORA-03113: end-of-file on communication channel
Process ID: 5280
Session ID: 5 Serial number: 3

As you can see database starts till MOUNT mode and then throws the error, meaning you cannot start the database in OPEN mode.
But you can start the database in MOUNT mode which is what we need to run the command. 

Step 5: Connect to sys since you have shut down the instance. It will connect to an idle instance.

SQL> CONN SYS AS SYSDBA
Enter password:
Connected to an idle instance.
Then run the following command

SQL> STARTUP MOUNT

Your database has started in MOUNT mode.


Step 6: Now run the following DBA command:

SQL> ALTER DATABASE CLEAR UNARCHIVED LOGFILE GROUP 6;


This is the correct syntax of the DBA's most dangerous command. The reason I have given GROUP 6 is because I have corrupted the redo log member of this group and no archive log of this redo log member is created since its status is ACTIVE.


Step 7: Check the status of your redo log file by running the following commands:

SQL> col member format a40
SQL> SELECT GROUP#,L.STATUS,V.MEMBER FROM V$LOG L JOIN V$LOGFILE V USING (GROUP#) ORDER BY GROUP#;

    GROUP# STATUS           MEMBER
---------- ---------------- ----------------------------------------
         1 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO04.LOG
         1 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO01.LOG
         2 CURRENT          C:\APP\PC\ORADATA\ADMIN\REDO02.LOG
         2 CURRENT          C:\APP\PC\ORADATA\ADMIN\REDO05.LOG
         3 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO03.LOG
         3 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO08.LOG
         4 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO06.LOG
         4 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO09.LOG
         5 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO10.LOG
         5 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO07.LOG
         6 UNUSED           C:\APP\PC\ORADATA\ADMIN\REDO_G6_M1.LOG


As you can see the status of the group which we have cleared is now UNUSED which is group 6 in this case.
Now let’s run the same query but this time we will check sequence of the group member too.

SQL> SELECT GROUP#,L.STATUS,V.MEMBER,L.SEQUENCE# FROM V$LOG L JOIN V$LOGFILE V USING (GROUP#) ORDER BY GROUP#;

    GROUP# STATUS           MEMBER                                    SEQUENCE#
---------- ---------------- ---------------------------------------- ----------
         1 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO04.LOG             2655
         1 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO01.LOG             2655
         2 CURRENT          C:\APP\PC\ORADATA\ADMIN\REDO02.LOG             2656
         2 CURRENT          C:\APP\PC\ORADATA\ADMIN\REDO05.LOG             2656
         3 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO03.LOG             2651
         3 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO08.LOG             2651
         4 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO06.LOG             2654
         4 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO09.LOG             2654
         5 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO10.LOG             2653
         5 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO07.LOG             2653
         6 UNUSED           C:\APP\PC\ORADATA\ADMIN\REDO_G6_M1.LOG            0


And as you can see the sequence of the UNUSED redo log group is 0.

Step 8: Now we will run switch log file command to change the current status which is UNUSED.  

SQL> ALTER SYSTEM SWITCH LOGFILE;
ALTER SYSTEM SWITCH LOGFILE
*
ERROR at line 1:
ORA-01109: database not open


OOPSSS !!! Looks like our database is not in OPEN mode. So to change the mode to OPEN mode run the following command:
 

SQL> ALTER DATABASE OPEN;

Database altered.

Now we will try to switch log file again.

SQL> ALTER SYSTEM SWITCH LOGFILE;

System altered.

Thus we need the database to be in OPEN mode to execute the above command.

Let's check the status and sequence of the redo log file again by executing the following command:

SQL> SELECT GROUP#,L.STATUS,V.MEMBER,L.SEQUENCE# FROM V$LOG L JOIN V$LOGFILE V USING (GROUP#) ORDER BY GROUP#;

    GROUP# STATUS           MEMBER                                    SEQUENCE#
---------- ---------------- ---------------------------------------- ----------
         1 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO04.LOG             2655
         1 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO01.LOG             2655
         2 ACTIVE           C:\APP\PC\ORADATA\ADMIN\REDO02.LOG             2656
         2 ACTIVE           C:\APP\PC\ORADATA\ADMIN\REDO05.LOG             2656
         3 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO03.LOG             2651
         3 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO08.LOG             2651
         4 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO06.LOG             2654
         4 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO09.LOG             2654
         5 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO10.LOG             2653
         5 INACTIVE         C:\APP\PC\ORADATA\ADMIN\REDO07.LOG             2653
         6 CURRENT          C:\APP\PC\ORADATA\ADMIN\REDO_G6_M1.LOG         2657


As you can see the status of the redo log file of group 6 has now changed to CURRENT from UNUSED and we also have new sequence generated which is 2657. 


Step 9: Then run the following command:

SELECT NAME,SEQUENCE# FROM V$ARCHIVED_LOG;

NAME                                                                                 SEQUENCE#
----------------------------------------------------------------------------------- ----------
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2562_CRVLOCS1_.ARC       2562
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2563_CRVLOHPY_.ARC       2563
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2564_CRVLOLMC_.ARC       2564
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2565_CRVLOWMH_.ARC       2565
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2566_CRVLP90D_.ARC       2566
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2567_CRVLPN4K_.ARC       2567
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2568_CRVLPR5M_.ARC       2568
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2569_CRVLPWPD_.ARC       2569
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2570_CRVLPY3R_.ARC       2570
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2571_CRVLQ7FM_.ARC       2571
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2572_CRVLQLNG_.ARC       2572
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2573_CRVLQXHO_.ARC       2573
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2574_CRVLR1RO_.ARC       2574
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2575_CRVLR61X_.ARC       2575
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2576_CRVLR83H_.ARC       2576
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2577_CRVLRJF6_.ARC       2577
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2578_CRVLRTWS_.ARC       2578
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2579_CRVLS4Q5_.ARC       2579
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2580_CRVLS9MP_.ARC       2580
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2581_CRVLSFGL_.ARC       2581
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2582_CRVLSGTB_.ARC       2582
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2583_CRVLSQW8_.ARC       2583
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2584_CRVLT1VN_.ARC       2584
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2585_CRVLTCNX_.ARC       2585
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2586_CRVLTHL1_.ARC       2586
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2587_CRVLTNGL_.ARC       2587
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2588_CRVLTOWF_.ARC       2588
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2589_CRVLTYVV_.ARC       2589
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2590_CRVLVC4H_.ARC       2590
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2591_CRVLVOVJ_.ARC       2591
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2592_CRVLVV03_.ARC       2592
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2593_CRVLVZD6_.ARC       2593
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2594_CRVLW126_.ARC       2594
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2595_CRVLWB02_.ARC       2595
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2596_CRVLWNWX_.ARC       2596
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2597_CRVLWZHC_.ARC       2597
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2598_CRVLX40J_.ARC       2598
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2599_CRVLX82F_.ARC       2599
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2600_CRVLXDKF_.ARC       2600
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2601_CRVLXN9Q_.ARC       2601
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2602_CRVLXZ8B_.ARC       2602
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2603_CRVLYCJ5_.ARC       2603
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2604_CRVNPR74_.ARC       2604
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_19\O1_MF_1_2605_CRW0BDTD_.ARC       2605
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_20\O1_MF_1_2606_CRXWNLM6_.ARC       2606
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_20\O1_MF_1_2607_CRYHXFPX_.ARC       2607
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_20\O1_MF_1_2608_CRZJ9D21_.ARC       2608
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_20\O1_MF_1_2609_CRZJDDFS_.ARC       2609
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_20\O1_MF_1_2610_CRZJFC7X_.ARC       2610
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_20\O1_MF_1_2611_CRZJJ5O0_.ARC       2611
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_21\O1_MF_1_2612_CRZLKMQJ_.ARC       2612
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_21\O1_MF_1_2613_CS11CP8M_.ARC       2613
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_22\O1_MF_1_2614_CS2W1B67_.ARC       2614
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_22\O1_MF_1_2615_CS3Z65M5_.ARC       2615
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_22\O1_MF_1_2616_CS44H33S_.ARC       2616
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_23\O1_MF_1_2617_CS5DKMGG_.ARC       2617
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_23\O1_MF_1_2618_CS5DKWJF_.ARC       2618
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_23\O1_MF_1_2619_CS5GPCQ7_.ARC       2619
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_23\O1_MF_1_2620_CS5X658D_.ARC       2620
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_23\O1_MF_1_2621_CS6FN2H1_.ARC       2621
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_23\O1_MF_1_2622_CS6M0VFG_.ARC       2622
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_24\O1_MF_1_2623_CS88S17B_.ARC       2623
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_24\O1_MF_1_2624_CS88SW50_.ARC       2624
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_24\O1_MF_1_2625_CS8GJH4H_.ARC       2625
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_24\O1_MF_1_2626_CS9SK8QP_.ARC       2626
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_25\O1_MF_1_2627_CSC737YB_.ARC       2627
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_25\O1_MF_1_2628_CSC8ZJKY_.ARC       2628
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_25\O1_MF_1_2629_CSCDZT73_.ARC       2629
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_25\O1_MF_1_2630_CSCKPPZ4_.ARC       2630
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_25\O1_MF_1_2631_CSDHOK8B_.ARC       2631
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_26\O1_MF_1_2632_CSFDNXLY_.ARC       2632
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_26\O1_MF_1_2633_CSG9PGK9_.ARC       2633
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_26\O1_MF_1_2634_CSGHJFY7_.ARC       2634
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_26\O1_MF_1_2635_CSGZJ3C2_.ARC       2635
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_27\O1_MF_1_2636_CSHZFMH5_.ARC       2636
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_27\O1_MF_1_2637_CSHZJ3KZ_.ARC       2637
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_27\O1_MF_1_2638_CSK6MJ86_.ARC       2638
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_27\O1_MF_1_2639_CSKRLT3J_.ARC       2639
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_27\O1_MF_1_2640_CSKRN617_.ARC       2640
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_27\O1_MF_1_2641_CSKROB4J_.ARC       2641
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_27\O1_MF_1_2642_CSKRONDX_.ARC       2642
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_28\O1_MF_1_2643_CSMH4LJZ_.ARC       2643
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_29\O1_MF_1_2644_CSOMZ97L_.ARC       2644
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_29\O1_MF_1_2645_CSOZQTMC_.ARC       2645
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_30\O1_MF_1_2646_CSR7YPD8_.ARC       2646
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_30\O1_MF_1_2647_CSR7ZFRL_.ARC       2647
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_30\O1_MF_1_2648_CSR805F0_.ARC       2648
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_30\O1_MF_1_2649_CSR85T4O_.ARC       2649
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_30\O1_MF_1_2650_CSRNXC30_.ARC       2650
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_30\O1_MF_1_2651_CSRNXGBP_.ARC       2651
                                                                                          2652
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_30\O1_MF_1_2655_CSSDML3L_.ARC       2655
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_30\O1_MF_1_2654_CSSDMLDZ_.ARC       2654
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_30\O1_MF_1_2653_CSSDMMX4_.ARC       2653
C:\APP\PC\FAST_RECOVERY_AREA\ADMIN\ARCHIVELOG\2016_07_30\O1_MF_1_2656_CSSGHRQT_.ARC       2656


You will get a long list. Just scroll to the bottom till you see a certain sequence whose name is nothing but a blank space. In this case look for sequence# 2652. You will find a blank space before it. The reason for the blank space is that there was no archive log created of that particular logfile and hence we had to run clear unarchived logfile command.

Now think carefully about what will happen if a situation arises where you are required to recover your database and restore it and you don’t have the backup after executing clear unarchived log file command?

Your backup, recovery and restore will work fine till the sequence# 2651 but when it reaches sequence# 2652 there will be a big mess. Since the redo log file wasn’t archived you will lose the data from that redo log file and we have seen earlier that we need redo log file for recovery in case of media failure. And in this case since we have no archive log of this redo log file we won’t be able to recover the database after sequence# 2651.

Thus TAKE BACKUP of the whole database as soon as you execute the following command and have followed the above steps:

SQL> ALTER DATABASE CLEAR UNARCHIVED LOGFILE;
