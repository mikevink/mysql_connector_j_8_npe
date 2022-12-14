# Sample project to highlight MySQL Connector/J 8.0.30 bug

## tl;dr

When the MySQL server dies while queries are being run, the `MySQL Connector/J 8.0.30` occasionally
fails with a NullPointerException (NPE), rather than a more useful CommunicationsException.

```java
java.lang.NullPointerException
    at com.mysql.cj.AbstractQuery.stopQueryTimer(AbstractQuery.java:228)
    at com.mysql.cj.jdbc.StatementImpl.stopQueryTimer(StatementImpl.java:643)
    at com.mysql.cj.jdbc.StatementImpl.executeQuery(StatementImpl.java:1182)
    at org.apache.commons.dbcp2.DelegatingStatement.executeQuery(DelegatingStatement.java:329)
    at org.apache.commons.dbcp2.DelegatingStatement.executeQuery(DelegatingStatement.java:329)
    at org.example.db.Dao.count(Dao.java:89)
    at org.example.Job.run(Job.java:52)
```

We believe this is caused by `AbstractQuery::closeQuery` being called before
`AbstractQuery::stopQueryTimer`.

This causes a problem for us as we cannot distinguish the NPE from any other non-sql errors and
thus we no longer attempt to reconnect after a backoff period.

When running the same code with `MySQL Connector/J 5.1.49`, all errors are handled as expected.

## Simple solution
The simplest solution would be to add a null-check to `AbstractQuery::stopQueryTimer`
```java
if (this.session != null) {
    this.session.getCancelTimer().purge();
}
```
This will solve the NPE symptom, though it would not prevent the `AbstractQuery::closeQuery` 
being called before `AbstractQuery::stopQueryTimer`, assuming that it is an undesired execution 
order.

## Requirements

* JDK 11
* local installation of MySQL server (tested with 5.7)
* 1 executable script that can start a MySQL server instance
* 1 executable script that can stop the MySQL server instance started above
* JetBrains IntelliJ

Note: it should be possible to run the program without IntelliJ, as long as you are familiar
enough with gradle.

## Setup

Before running the project, you need to update the `Main::mySQLController` method to point to the
correct start & stop scripts for a MySQL server.

```java
// example script configuration
final String homeDirectory=System.getProperty("user.home");
final String startScriptPath=String.format("%s/bin/startMySQL.sh",homeDirectory);
final String stopScriptPath=String.format("%s/bin/stopMySQL.sh",homeDirectory);
```

The sample program expects to connect to MySQL using the following configuration

```java
source.setUrl("jdbc:mysql://127.0.0.1:3306");
    source.setUsername("root");
    source.setPassword("");
```

If your server is set up differently, please update the `DSFactory::create` method as necessary.

The project uses the slf4j SimpleLogger, and thus can be configured using the system properties
specified on the [documentation page](https://www.slf4j.org/api/org/slf4j/impl/SimpleLogger.html)
. The default log level is `info`, with more information available at the `debug` and `trace`
levels.

## Running the project

The entry point for the project is `Main.java`.

Once the setup has been completed, running the project will:

1. Start a MySQL server
2. Create a JobRunner, which will spawn 30 Jobs and give them a ScheduledExecutor with 10
   parallel threads.
3. Initialise the DBs, cleaning up any leftover schemas and creating a sample table to be worked on
4. Instruct all the Jobs to schedule themselves
    * The Jobs will reschedule themselves on successful completion every 500 ms
5. Wait for all the jobs to have run at least once
6. Sleep for 5 seconds (may not be necessary)
7. Stop the MySQL server
8. Wait for all the jobs to terminate, one way or another
9. Print the last state of the jobs, to see which have handled the MySQL termination as a
   disconnection and which have not

## Results

Once the sample project stops the MySQL server, we would expect all the jobs to eventually
terminate with a `Disconnected` state. But what we see using the `8.0.30` version of the connector
is

```shell
# this is the last bit of the project output
2022-09-06 17:55:09.414 [main] INFO JobRunner - Disabled jobs (should be empty):
sqltest_1::Disabled
sqltest_2::Disabled
sqltest_3::Disabled
sqltest_5::Disabled
sqltest_8::Disabled
sqltest_10::Disabled
sqltest_13::Disabled
sqltest_14::Disabled
```

Running the same program, but with version `5.1.49` gets us the expected output of

```shell
2022-09-06 18:32:13.316 [main] INFO JobRunner - Disabled jobs (should be empty):

```

## Deep dive

Scrolling back through the console output of the original run (v. `8.0.30`) reveals that all the
disabled jobs ran into the NPE generated in `com.mysql.cj.AbstractQuery::stopQueryTimer` on line
228, which is:
```java
this.session.getCancelTimer().purge();
```
So, at the point of failure, either `this.session` is `null` or `NativeSession::getCancelTimer` 
returns `null`.

A quick look at the body of `NativeSession::getCancelTimer` shows that it can't be our culprit:
```java
public synchronized Timer getCancelTimer() {
    if (this.cancelTimer == null) {
        this.cancelTimer = new Timer("MySQL Statement Cancellation Timer", Boolean.TRUE);
    }
    return this.cancelTimer;
}
```
Which leaves us with `this.session`. It starts off as `null` on `AbstractQuery.java` line `53`
```java
public NativeSession session = null;
```
but it gets assigned a value in the class constructor. It's unlikely that it will be `null` 
there, but we can set a conditional breakpoint there to log a message should it ever get hit.
```yaml
breakpoint:
  location: AbstractQuery:97
  condition: null == sess
  suspend: disabled
  evaluate and log: 'System.currentTimeMillis() + " BREAKPOINT: " + this + " with null parameter || Stack: " + Arrays. toString(Thread.currentThread().getStackTrace())'
```

The variable also gets set to `null` in the `AbstractQuery::closeQuery` method, one line `158`:
```java
public void closeQuery() {
    this.queryAttributesBindings = null;
    this.session = null;
}
```
So we added another logging breakpoint in that method call.
```yaml
breakpoint:
   location: AbstractQuery:97
   suspend: disabled
   evaluate and log: 'System.currentTimeMillis() + " BREAKPOINT: closeQuery on " + this + " with " + this.session + " || Stack: " + Arrays.toString(Thread.currentThread().getStackTrace())'
```
For the sake of a complete image, we also added logging breakpoints to the 
`AbstractQuery::stopQueryTimer` method on lines `221` and `228` (wondering if there was perhaps 
a race condition whereby `AbstractQuery::closeQuery` was called in between the execution of 
these two lines).
```yaml
breakpoint221:
  location: AbstractQuery:221
  suspend: disabled
  evaluate and log: 'System.currentTimeMillis() + " BREAKPOINT: cancel on " + this + " with " + this.session + " and " + timeoutTask + " || Stack: " + Arrays.toString(Thread.currentThread().getStackTrace())'
breakpoint228:
  location: AbstractQuery:228
  suspend: disabled
  condition: null == this.session
  evaluate and log: 'System.currentTimeMillis() + " BREAKPOINT: NPE on " + this + " with " + timeoutTask + " || Stack: " + Arrays.toString(Thread.currentThread().getStackTrace())'
```
And one more in `AbstractQuery::startQueryTimer` on line `213` to track the creation of the 
`timeoutTask`:
```yaml
breakpoint:
   location: AbstractQuery:213
   suspend: disabled
   evaluate and log: 'System.currentTimeMillis() + " BREAKPOINT: startQueryTimer on " + this + " with " + this.session + " and " + timeoutTask + " || Stack: " + Arrays.toString(Thread.currentThread().getStackTrace())'
```
The full output of a Debug run with the above breakpoints can be found in 
`debug_with_extra_points.log`.

On lines 759-761 of the above file, we find the NPEs generated in this run. We can use the hash 
of the `com.mysql.cj.SimpleQuery` (which extends `AbstractQuery`)  classes to filter out only the 
log entries we want to have a look at
```shell
grep -E "32eef325|51ef6da5|57e94d97" debug_with_extra_points.log > npe_only.log
```

We notice that, in all 3 cases, the execution order is:
1. `AbstractQuery::startQueryTimer` ()
2. `AbstractQuery::closeQuery` (where `this.session` is set to `null`)
3. `AbstractQuery::stopQueryTimer` line 221 (`timeoutTask::cancel`)
4. `AbstractQuery::stopQueryTimer` line 228 (NPE caused by `this.session.getCancelTimer()`)


We choose the `32eef325` hash for a closer look
```shell
grep -E "32eef325" npe_only.log > npe_32eef325.log
```
and expand the stack trace generated by the extra debug logs.

For `AbstractQuery::closeQuery`
```java
com.mysql.cj.AbstractQuery.closeQuery(AbstractQuery.java:157)
com.mysql.cj.jdbc.StatementImpl.closeQuery(StatementImpl.java:2182)
com.mysql.cj.jdbc.StatementImpl.realClose(StatementImpl.java:1803)
com.mysql.cj.jdbc.ConnectionImpl.closeAllOpenStatements(ConnectionImpl.java:737)
com.mysql.cj.jdbc.ConnectionImpl.realClose(ConnectionImpl.java:1716)
com.mysql.cj.jdbc.ConnectionImpl.cleanup(ConnectionImpl.java:614)
com.mysql.cj.jdbc.ConnectionImpl.handleCleanup(ConnectionImpl.java:2689)
com.mysql.cj.NativeSession.invokeCleanupListeners(NativeSession.java:816)
com.mysql.cj.NativeSession.execSQL(NativeSession.java:685)
com.mysql.cj.jdbc.StatementImpl.executeQuery(StatementImpl.java:1168)
org.apache.commons.dbcp2.DelegatingStatement.executeQuery(DelegatingStatement.java:329)
org.apache.commons.dbcp2.DelegatingStatement.executeQuery(DelegatingStatement.java:329)
org.example.db.Dao.count(Dao.java:89)
org.example.Job.run(Job.java:52)
```

For `AbstractQuery::stopQueryTimer`
```java
com.mysql.cj.AbstractQuery.stopQueryTimer(AbstractQuery.java:221)
com.mysql.cj.jdbc.StatementImpl.stopQueryTimer(StatementImpl.java:643)
com.mysql.cj.jdbc.StatementImpl.executeQuery(StatementImpl.java:1182)
org.apache.commons.dbcp2.DelegatingStatement.executeQuery(DelegatingStatement.java:329)
org.apache.commons.dbcp2.DelegatingStatement.executeQuery(DelegatingStatement.java:329)
org.example.db.Dao.count(Dao.java:89)
org.example.Job.run(Job.java:52)
```

It appears that the `AbstractQuery` implementation has its inner query closed during the
`NativeSession` cleanup triggered by the server disconnect on line `684` (a
`CJCommunicationsException` in `NativeSennion::execSQL`), which makes sense. The StatementImpl,
however, isn't made aware of the communication exception, and it attempts to clean up the timers
it had previously started, which leads to the NPE.

# Final Notes

* There does not appear to be any branching in the code to indicate why the NPE isn't always 
triggered, as the execution path should be the same each time.
* Adding a null-check as suggested in __Simple Solution__ would prevent the error being raised. 
* The `Timer` which schedules the `CancelQueryTaskImpl` in `AbstractQuery::startQueryTImer` line 
  `213` does not receive the `purge()` call it should. As the task gets cancelled anyway, this 
  is likely not a problem, but we are not familiar enough with the rest of the code to be certain.