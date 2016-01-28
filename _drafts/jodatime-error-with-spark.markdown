---
layout: post
title: "Joda-Time Error With Spark"
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

# Version Conflicts

Although Maven successfully compiled the JAR, it doesn't mean the JAR is immune to issues.  When I run my code using `spark-submit` and look at the logs, I encounter:

    com.amazonaws.services.s3.model.AmazonS3Exception: AWS authentication requires a valid Date or x-amz-date header (Service: Amazon S3; Status Code: 403; Error Code: AccessDenied; Request ID: 54B9BA07F04C289F), S3 Extended Request ID: Qs4vHGItzXaKlRmSwuKBTSXKtEnOBOo+eIdsQ+iYr0HnK2PxP/byjI4+wCB66j2OEnrwEWT5qD4=

I google [!the error](http://stackoverflow.com/questions/32058431/aws-java-sdk-aws-authentication-requires-a-valid-date-or-x-amz-date-header), and it seems like I have an older Joda-Time dependency being pulled in somewhere in my JAR.  In order to view the dependency tree but only for the Joda-Time package, I run the following.

{% highlight bash %}
$ mvn dependency:tree -Dverbose -Dincludes="joda-time":"joda-time"

{% endhighlight %}





