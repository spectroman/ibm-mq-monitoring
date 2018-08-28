MQ Monitoring Framework
=======================

# Introduction

Here is the addendum to zabbix agent that is capable of adding monitoring on IBM MQ queue managers, channels and queues.

This is a fully automatic auto discovery system that will auto create a template perfectly tailored to what type of machine is configured there.

## Requirements

* Linux OS
* IBM MQ at least version 7.x.x

### Requirements from overall installation

You must add SUDO powers to the zabbix user on the target machine.  
Which will allow the zabbix agent to personify the IBM MQ user "mqm" and thus capable of executing the necessary MQ console commands accordingly.

At `/etc/sudoers` must be included :

`Defaults:zabbix !requiretty
zabbix  ALL=(ALL) NOPASSWD: /usr/bin/su`

# Inner working

### The problem

It is not possible in zabbix, at this moment, to discover something and use the value from this discovery in a second LLD (low level discovery), it is also not possible to use a user macro (*{$BLABLA}*) in a LLD, as such:

### The solution

- When playbook is attached to the host in question, a basic template must be attached to it in zabbix: **Template App IBM MQ**
- This template will run an LLD named: **Queue Manager Discovery** which contains the command:

`/etc/zabbix/zabbix_agentd.d/mq_framework/qm-discovery d`

The return will be a JSON for LLD:

`
[root@mq-app-t15 mq_framework]# ./qm-discovery d
{ "data": [{ "{#QMNAME}" : "QMAPPT15"}]}
`

- Along this run, he will run the script **zabbix-api** which will create dynamically a second template attached on this host.
- The new template will contain commands to discover the Channels and the Queues from the previously referred Queue Manager name.
- The variable *mq_priority* will give the ability to generate a template named for DTA (Developtment, Test or Acceptance) or PM (Production and Management)
  - `mq_priority:5` will create template with PM type and all the triggers will be set to **DISASTER**
  - `mq_priority:4` or any number smaller than 5 will create triggers with such priority index and the name will be followed by *_DTA* suffix
- The default variables in *defaults/main.yml* are used specifically to configure the *zabbix-api* script;
- Some of the queues actually have a different logic than the majority, where most cannot remain above 0, some cannot remain at 0 and others cannot generate alerts at night, we generally know their name, as such:
  - `mq_inverse_queues:` is an array of queue names that will be descriminated during the creation of the new template and static items will be put into place. And the triggers for these queues will have the logic inverse, meaning they will alert if they are ZERO for as long as one hour or so;
  - `mq_daytime_triggers:` is an array of queue names that will be descriminated during the creation of the new template and static items will be put into place. And the triggers for these queues have the addition to only generate alarms between 0800 and 2200 hours;
  - The items on these lists must be the same list of queues described in the regular expression that must follow the zabbix installation using this methods.
    - For it all work and the template be successfully used on zabbix, a regular expression must be created that contains the following, for exemple:
      - 1    »    QL.SOME.FF.MUTEX    [Character string not included]
      - 2    »    NET.FF.CONNECTORMUTEXQ    [Character string not included]
- The name of such regular expression must match with the variable: `mq_zabbix_regex_name`

# The Conclusion

This will provide the initial necessary overview for an IBM MQ, but then:

- The command which reads channels and queues, is a wrapper to run commands inside IBM MQ Console, as such, any command and value can be extracted from MQ with a simple command:
  - display-wrapper QUEUEMANAGER QUEUE|CHANNEL DISPLAY_TYPE PROPERTY|ALL
  - a better evaluation can be see as follow:

```
[root@mq-app-t15 mq_framework]# ./display-wrapper QMAPPT15 TI.LOG QUEUE ALL
5724-H72 (C) Copyright IBM Corp. 1994, 2011.  ALL RIGHTS RESERVED.
Starting MQSC for queue manager QMAPPT15.
     1 : DISPLAY QUEUE(TI.LOG)
AMQ8409: Display Queue details.
   QUEUE(TI.LOG)                           TYPE(QLOCAL)
   ACCTQ(QMGR)                             ALTDATE(2017-05-09)
   ALTTIME(18.07.00)                       BOQNAME( )
   BOTHRESH(0)                             CLUSNL( )
   CLUSTER( )                              CLCHNAME( )
   CLWLPRTY(0)                             CLWLRANK(0)
   CLWLUSEQ(QMGR)                          CRDATE(2017-05-09)
   CRTIME(18.07.00)                        CURDEPTH(0)
   CUSTOM( )                               DEFBIND(OPEN)
   DEFPRTY(0)                              DEFPSIST(NO)
   DEFPRESP(SYNC)                          DEFREADA(NO)
   DEFSOPT(SHARED)                         DEFTYPE(PREDEFINED)
   DESCR(no description)                   DISTL(NO)
   GET(ENABLED)                            HARDENBO
   INITQ( )                                IPPROCS(0)
   MAXDEPTH(5000)                          MAXMSGL(4194304)
   MONQ(QMGR)                              MSGDLVSQ(PRIORITY)
   NOTRIGGER                               NPMCLASS(NORMAL)
   OPPROCS(0)                              PROCESS( )
   PUT(ENABLED)                            PROPCTL(COMPAT)
   QDEPTHHI(80)                            QDEPTHLO(20)
   QDPHIEV(DISABLED)                       QDPLOEV(DISABLED)
   QDPMAXEV(ENABLED)                       QSVCIEV(NONE)
   QSVCINT(999999999)                      RETINTVL(999999999)
   SCOPE(QMGR)                             SHARE
   STATQ(QMGR)                             TRIGDATA( )
   TRIGDPTH(1)                             TRIGMPRI(0)
   TRIGTYPE(FIRST)                         USAGE(NORMAL)
One MQSC command read.
No commands have a syntax error.
All valid MQSC commands were processed.
[root@mq-app-t15 mq_framework]# ./display-wrapper QMAPPT15 TI.LOG QUEUE CURDEPTH
0
[root@mq-app-t15 mq_framework]# ./display-wrapper QMAPPT15 TI.LOG QUEUE TYPE
QLOCAL
[root@mq-app-t15 mq_framework]# ./display-wrapper QMAPPT15 TI.LOG QSTATUS ALL
5724-H72 (C) Copyright IBM Corp. 1994, 2011.  ALL RIGHTS RESERVED.
Starting MQSC for queue manager QMAPPT15.
     1 : DISPLAY QSTATUS(TI.LOG)
AMQ8450: Display queue status details.
   QUEUE(TI.LOG)                           TYPE(QUEUE)
   CURDEPTH(0)                             IPPROCS(0)
   LGETDATE( )                             LGETTIME( )
   LPUTDATE( )                             LPUTTIME( )
   MEDIALOG( )                             MONQ(OFF)
   MSGAGE( )                               OPPROCS(0)
   QTIME( , )                              UNCOM(NO)
One MQSC command read.
No commands have a syntax error.
All valid MQSC commands were processed.
[root@mq-app-t15 mq_framework]# ./display-wrapper QMAPPT15 TI.LOG QSTATUS MONQ
OFF
[root@mq-app-t15 mq_framework]#
```
