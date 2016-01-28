---
layout: post
title: "Dependency Management With Spark"
---

You are probably familiar with tools like Maven or sbt build tools to manage your dependencies, but sometimes dependency management can get more complicated in Spark.  In this post, I'm going to use the the Joda-Time library as an example issue I ran into when working with Spark dependencies.  For the time being, I'm not going to discuss packaging dependencies for Python projects because that works differently from Java/Scala projects.

*Note: The example problem described below is not applicable for the latest Spark version 1.6.  The issue was resolved in https://issues.apache.org/jira/browse/SPARK-11413*

# Dependencies

In Maven, your dependencies are managed in a pom.xml file.  It will look something like this:

{% highlight xml %}
<dependencies>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-core_2.10</artifactId>
        <version>1.5.0</version>
        <scope>provided</scope>
    </dependency>

    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-streaming_2.10</artifactId>
        <version>1.5.0</version>
        <scope>provided</scope>
    </dependency>

    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-streaming-kinesis-asl_2.10</artifactId>
        <version>1.5.0</version>
    </dependency>

    <dependency>
        <groupId>com.amazonaws</groupId>
        <artifactId>aws-java-sdk-s3</artifactId>
        <version>1.10.20</version>
    </dependency>
    
    <!-- more dependencies... -->

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>2.3</version>
                <configuration>
                    <downloadSources>true</downloadSources>
                    <downloadJavadocs>false</downloadJavadocs>
                    <filters>
                        <filter>
                            <artifact>*:*</artifact>
                            <excludes>
                                <exclude>META-INF/*.SF</exclude>
                                <exclude>META-INF/*.DSA</exclude>
                                <exclude>META-INF/*.RSA</exclude>
                            </excludes>
                        </filter>
                    </filters>
                    <transformers>
                        <transformer
                            implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                            <resource>reference.conf</resource>
                        </transformer>
                    </transformers>
                </configuration>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

            <!-- more plugins... -->
        
        </plugins>
    </build>
</dependencies>
{% endhighlight %}

spark-core and spark-streaming are listed as `provided` because they are provided by the cluster managers [!at run time](http://spark.apache.org/docs/latest/submitting-applications.html).  In my code, I connect to an Amazon S3 bucket as well as to a Kinesis stream, so I have dependencies listed for those libraries.  Additionally I need to use the Maven shade plugin because I want an [!uber jar](https://maven.apache.org/plugins/maven-shade-plugin/) that contains not only my class files, but also the class files of the dependencies.  The reason is that the driver and executors in Spark need access to all the class files in order to successfully execute the job, and the easiest way to do that is to package everything up in one big JAR.

I now package my code into a JAR using `mvn package`, and if all goes well, it will show:

    [INFO] ------------------------------------------------------------------------
    [INFO] BUILD SUCCESS
    [INFO] ------------------------------------------------------------------------
    [INFO] Total time: 18.872 s
    [INFO] Finished at: 2016-01-27T16:58:00-08:00
    [INFO] Final Memory: 123M/610M
    [INFO] ------------------------------------------------------------------------    

# Where is the error?

Although Maven successfully compiled the JAR, the execution of the code can fail at run-time.  When I run my code using `spark-submit` and look at the logs, I encounter:

    com.amazonaws.services.s3.model.AmazonS3Exception: AWS authentication requires a valid Date or x-amz-date header (Service: Amazon S3; Status Code: 403; Error Code: AccessDenied; Request ID: 54B9BA07F04C289F), S3 Extended Request ID: Qs4vHGItzXaKlRmSwuKBTSXKtEnOBOo+eIdsQ+iYr0HnK2PxP/byjI4+wCB66j2OEnrwEWT5qD4=

I google [!the error](http://stackoverflow.com/questions/32058431/aws-java-sdk-aws-authentication-requires-a-valid-date-or-x-amz-date-header), and it seems like I have an older Joda-Time dependency being pulled in somewhere in my JAR.  In order to view the dependency tree but only for the Joda-Time package, I run the following.


    $ mvn dependency:tree -Dverbose -Dincludes="joda-time":"joda-time"

    [INFO] ------------------------------------------------------------------------
    [INFO] Building Demo 0.0.1-SNAPSHOT
    [INFO] ------------------------------------------------------------------------
    [INFO] 
    [INFO] --- maven-dependency-plugin:2.8:tree (default-cli) @ DemoModel ---
    [INFO] com.example:Demo:jar:0.0.1-SNAPSHOT
    [INFO] \- com.amazonaws:amazon-kinesis-client:jar:1.6.1:compile
    [INFO]    \- com.amazonaws:aws-java-sdk-core:jar:1.10.20:compile
    [INFO]       \- joda-time:joda-time:jar:2.8.1:compile
    [INFO] ------------------------------------------------------------------------
    [INFO] BUILD SUCCESS
    [INFO] ------------------------------------------------------------------------
    [INFO] Total time: 6.134 s
    [INFO] Finished at: 2016-01-28T13:37:04-08:00
    [INFO] Final Memory: 29M/513M
    [INFO] ------------------------------------------------------------------------

This is where confusion ensues.  Normally we would have expected the error to be an older version of Joda-Time that was compiled, and then we can fix that by including an `<exclusion>` tag or explicitly listing joda-time as its own top-level dependency in the pom.xml.  But here, Maven tells us that the version for Joda-Time is in fact 2.8.1, the exact version that we need.  So how can we still get the error above?

If you encounter this type of issue, I recommend determining where the class in question is [!being loaded from at run-time](http://stackoverflow.com/questions/947182/how-to-explore-which-classes-are-loaded-from-which-jars).  With `spark-submit`, you can specify extra Java options with:

{% highlight bash %}
./bin/spark-submit --master yarn-cluster --class <your-class-entrypoint> --conf "spark.executor.extraJavaOptions=-verbose:class" --conf "spark.driver.extraJavaOptions=-verbose:class" myApp.jar
{% endhighlight %}

You can also specify these configurations in the conf/spark-defaults.conf file like this:

    spark.driver.extraJavaOptions="-verbose:class"
    spark.executors.extraJavaOptions="-verbose:class"

Run the Spark job, inspect the logs, and you'll find something like the following:

    [Loaded org.joda.time.ReadableInstant from file:/tmp/hadoop-hduser/nm-local-dir/filecache/10/spark-assembly-1.5.0-hadoop2.6.0.jar]
    [Loaded org.joda.time.DateTimeZone from file:/tmp/hadoop-hduser/nm-local-dir/filecache/10/spark-assembly-1.5.0-hadoop2.6.0.jar]
    [Loaded org.joda.time.tz.FixedDateTimeZone from file:/tmp/hadoop-hduser/nm-local-dir/filecache/10/spark-assembly-1.5.0-hadoop2.6.0.jar]
    [Loaded org.joda.time.JodaTimePermission from file:/tmp/hadoop-hduser/nm-local-dir/filecache/10/spark-assembly-1.5.0-hadoop2.6.0.jar]
    [Loaded org.joda.time.Chronology from file:/tmp/hadoop-hduser/nm-local-dir/filecache/10/spark-assembly-1.5.0-hadoop2.6.0.jar]
    [Loaded org.joda.time.chrono.BaseChronology from file:/tmp/hadoop-hduser/nm-local-dir/filecache/10/spark-assembly-1.5.0-hadoop2.6.0.jar]
    [Loaded org.joda.time.DateTimeZone$1 from file:/tmp/hadoop-hduser/nm-local-dir/filecache/10/spark-assembly-1.5.0-hadoop2.6.0.jar]
    [Loaded org.joda.time.IllegalInstantException from file:/tmp/hadoop-hduser/nm-local-dir/filecache/10/spark-assembly-1.5.0-hadoop2.6.0.jar]
    [Loaded org.joda.time.tz.NameProvider from file:/tmp/hadoop-hduser/nm-local-dir/filecache/10/spark-assembly-1.5.0-hadoop2.6.0.jar]
    [Loaded org.joda.time.tz.Provider from file:/tmp/hadoop-hduser/nm-local-dir/filecache/10/spark-assembly-1.5.0-hadoop2.6.0.jar]

The issue is that the Spark assembly JAR is also compiled with the Joda-Time dependency, and instead of pulling the Joda-Time class from our application JAR at run-time, it pulls the Joda-Time class from Spark's JAR.  Prior to Spark 1.6, the version of Joda-Time compiled with Spark was 2.5, which is the reason we encounter the error **even though our application JAR contains the correct version of Joda-Time**.

# The fix

One solution is to use Spark 1.6, or rebuild the Spark assembly JAR with:

{% highlight bash %}
build/mvn -Djoda.version=2.9 -Phadoop-2.6 -Dhadoop.version=2.6.0 -DskipTests clean package

{% endhighlight %}

There is also another option to use the experimental configuration [!userClassPathFirst](https://spark.apache.org/docs/latest/configuration.html).  Specify in `spark-submit` or in the conf/spark-defaults.conf file the following:

    spark.driver.userClassPathFirst true
    spark.executor.userClassPathFirst true

This is an experimental feature that gives precedence to user-added JARs over Spark for classes loaded in the driver and executor.  This means it will load the Joda-Time class from the application JAR instead of loading the Joda-Time class from Spark's JAR, but the caveat is that the classes used in the application JAR can now crash Spark.  Unfortunately that's exactly what happened in my case.

    INFO yarn.YarnAllocator: Container marked as failed: container_1453439488983_0138_02_000002. Exit status: 1. Diagnostics: Exception from container-launch.
    Container id: container_1453439488983_0138_02_000002
    Exit code: 1
    Stack trace: ExitCodeException exitCode=1: 
        at org.apache.hadoop.util.Shell.runCommand(Shell.java:538)
        at org.apache.hadoop.util.Shell.run(Shell.java:455)
        at org.apache.hadoop.util.Shell$ShellCommandExecutor.execute(Shell.java:715)
        at org.apache.hadoop.yarn.server.nodemanager.DefaultContainerExecutor.launchContainer(DefaultContainerExecutor.java:211)
        at org.apache.hadoop.yarn.server.nodemanager.containermanager.launcher.ContainerLaunch.call(ContainerLaunch.java:302)
        at org.apache.hadoop.yarn.server.nodemanager.containermanager.launcher.ContainerLaunch.call(ContainerLaunch.java:82)
        at java.util.concurrent.FutureTask.run(FutureTask.java:266)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)

What else can we do?  The last solution I used was to download the Joda-Time JAR on every node and specify the JAR in the extra classpath configuration in Spark.

    spark.driver.extraClassPath /home/hduser/joda-time-2.8.1/joda-time-2.8.1.jar
    spark.executor.extraClassPath /home/hduser/joda-time-2.8.1/joda-time-2.8.1.jar

Running the Spark job this way, we can clearly see the correct version of Joda-Time loaded at run-time 

    [Loaded org.joda.time.ReadableInstant from file:/home/hduser/joda-time-2.8.1/joda-time-2.8.1.jar]
    [Loaded org.joda.time.DateTimeZone from file:/home/hduser/joda-time-2.8.1/joda-time-2.8.1.jar]
    [Loaded org.joda.time.tz.FixedDateTimeZone from file:/home/hduser/joda-time-2.8.1/joda-time-2.8.1.jar]
    [Loaded org.joda.time.JodaTimePermission from file:/home/hduser/joda-time-2.8.1/joda-time-2.8.1.jar]
    [Loaded org.joda.time.IllegalInstantException from file:/home/hduser/joda-time-2.8.1/joda-time-2.8.1.jar]
    [Loaded org.joda.time.tz.NameProvider from file:/home/hduser/joda-time-2.8.1/joda-time-2.8.1.jar]
    [Loaded org.joda.time.tz.Provider from file:/home/hduser/joda-time-2.8.1/joda-time-2.8.1.jar]

# Conclusion

Managing dependencies can be a pain, but hopefully this gives you some idea of how to tackle similar issues.  Make sure you don't have any version conflicts with dependencies in the application JAR, and then ensure the class is loaded from the application JAR as you expect, not another version from somewhere else.

