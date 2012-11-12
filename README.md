% Hadoop Quick Start - MongoDB
% 
% 

Quick Search

[![mongoDB](./Hadoop%20Quick%20Start%20-%20MongoDB_files/logo-mongoDB.png)](http://www.mongodb.org/)

5.  Hadoop Quick Start

[Hadoop Quick Start](./Hadoop%20Quick%20Start%20-%20MongoDB_files/Hadoop%20Quick%20Start%20-%20MongoDB.html)
============================================================================================================

[Prerequisites](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#HadoopQuickStart-Prerequisites)

-   [Hadoop](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#HadoopQuickStart-Hadoop)
-   [MongoDB](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#HadoopQuickStart-MongoDB)
-   [Miscelllaneous](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#HadoopQuickStart-Miscelllaneous)

[Building MongoDB
Adapter](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#HadoopQuickStart-BuildingMongoDBAdapter)

[Examples](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#HadoopQuickStart-Examples)

-   [Load Sample
    Data](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#HadoopQuickStart-LoadSampleData)
-   [Treasury
    Yield](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#HadoopQuickStart-TreasuryYield)
-   [UFO
    Sightings](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#HadoopQuickStart-UFOSightings)

MongoDB and Hadoop are a powerful combination and can be used together
to deliver complex analytics and data processing for data stored in
MongoDB. The following guide shows how you can start working with the
MongoDB-Hadoop adapter. Once you become familiar with the adapter, you
can use it to pull your MongoDB data into Hadoop Map-Reduce jobs,
process the data and return results back to a MongoDB collection.

### Prerequisites

#### Hadoop

In order to use the following guide, you should already have Hadoop up
and running. This can range from a deployed cluster containing multiple
nodes or a single node pseudo-distributed Hadoop installation running
locally. As long as you are able to run any of the examples on your
Hadoop installation, you should be all set. The following versions of
Hadoop are currently supported:

-   0.20/0.20.x
-   1.0/1.0.x
-   0.21/0.21.x
-   CDH3
-   CDH4

#### MongoDB

The latest version of MongoDB should be installed and running. In
addition, the MongoDB commands should be in your `$PATH`.

#### Miscelllaneous

In addition to Hadoop, you should also have `git` and JDK 1.6 installed.

### Building MongoDB Adapter

The MongoDB-Hadoop adapter source is available on github. First, clone
the repository and get the `release-1.0` branch:

~~~~ {.code-java}
$ git clone https://github.com/mongodb/mongo-hadoop.git
$ git checkout release-1.0
~~~~

Now, edit `build.sbt` and update the build target in
`hadoopRelease in ThisBuild`. In this example, we’re using the CDH3
Hadoop distribution from Cloudera so I’ll set it as follows:

~~~~ {.code-java}
hadoopRelease in ThisBuild := "cdh3"
~~~~

To build the adapter, use the self-bootstrapping version of `sbt` that
ships with the MongoDB-Hadoop adapter:

~~~~ {.code-java}
$ ./sbt package
~~~~

Once the adapter is built, you will need to copy it and the latest
stable version of the [MongoDB Java
driver](https://github.com/mongodb/mongo-java-driver/downloads) to your
`$HADOOP_HOME/lib` directory. For example, if you have Hadoop installed
in `/usr/lib/hadoop`:

~~~~ {.code-java}
$ wget --no-check-certificate https://github.com/downloads/mongodb/mongo-java-driver/mongo-2.7.3.jar
$ cp mongo-2.7.3.jar /usr/lib/hadoop/lib/
$ cp core/target/mongo-hadoop-core_cdh3u3-1.0.0.jar /usr/lib/hadoop/lib/
~~~~

### Examples

#### Load Sample Data

The MongoDB-Hadoop adapter ships with a few examples of how to use the
adapter in your own setup. In this guide, we’ll focus on the UFO
Sightings and Treasury Yield examples. To get started, first load the
sample data for these examples:

~~~~ {.code-java}
$ ./sbt load-sample-data
~~~~

To confirm that the sample data was loaded, start the `mongo` client and
look for the `mongo_hadoop` database and be sure that it contains the
`ufo_sightings.in` and `yield_historical.in` collections:

~~~~ {.code-java}
$ mongo
MongoDB shell version: 2.0.5
connecting to: test
> show dbs
mongo_hadoop    0.453125GB
> use mongo_hadoop
switched to db mongo_hadoop
> show collections
system.indexes
ufo_sightings.in
yield_historical.in
~~~~

#### Treasury Yield

To build the Treasury Yield example, we’ll need to first edit one of the
configuration files uses by the example code and set the MongoDB
location for the input (`mongo.input.uri`) and output
(`mongo.output.uri`) collections (in this example, Hadoop is running on
a single node alongside MongoDB). :

~~~~ {.code-java}
$ emacs examples/treasury_yield/src/main/resources/mongo-treasury_yield.xml
...
  <property>
    <!-- If you are reading from mongo, the URI -->
    <name>mongo.input.uri</name>
    <value>mongodb://127.0.0.1/mongo_hadoop.yield_historical.in</value>
  </property>
  <property>
    <!-- If you are writing to mongo, the URI -->
    <name>mongo.output.uri</name>
    <value>mongodb://127.0.0.1/mongo_hadoop.yield_historical.out</value>
  </property>
...
~~~~

Next, edit the main class that we’ll use for our MapReduce job
(`TreasuryYieldXMLConfig.java`) and update the class definition as
follows:

~~~~ {.code-java}
$ emacs examples/treasury_yield/src/main/java/com/mongodb/hadoop/examples/treasury/TreasuryYieldXMLConfig.java
...
public class TreasuryYieldXMLConfig extends MongoTool {

    static{
        // Load the XML config defined in hadoop-local.xml
        // Configuration.addDefaultResource( "hadoop-local.xml" );
        Configuration.addDefaultResource( "mongo-defaults.xml" );
        Configuration.addDefaultResource( "mongo-treasury_yield.xml" );
    }

    public static void main( final String[] pArgs ) throws Exception{
        System.exit( ToolRunner.run( new TreasuryYieldXMLConfig(), pArgs ) );
    }
}
...
~~~~

Now let’s build the Treasury Yield example:

~~~~ {.code-java}
$ ./sbt treasury-example/package
~~~~

Once the example is done building we can submit our MapReduce job:

~~~~ {.code-java}
$ hadoop jar examples/treasury_yield/target/treasury-example_cdh3u3-1.0.0.jar com.mongodb.hadoop.examples.treasury.TreasuryYieldXMLConfig
~~~~

This job should only take a few moments as it’s a relatively small
amount of data. Now check the output collection data in MongoDB to
confirm that the MapReduce job was successful:

~~~~ {.code-java}
$ mongo
MongoDB shell version: 2.0.5
connecting to: test
> use mongo_hadoop
switched to db mongo_hadoop
> db.yield_historical.out.find()
{ "_id" : 1990, "value" : 8.552400000000002 }
{ "_id" : 1991, "value" : 7.8623600000000025 }
{ "_id" : 1992, "value" : 7.008844621513946 }
{ "_id" : 1993, "value" : 5.866279999999999 }
{ "_id" : 1994, "value" : 7.085180722891565 }
{ "_id" : 1995, "value" : 6.573920000000002 }
{ "_id" : 1996, "value" : 6.443531746031742 }
{ "_id" : 1997, "value" : 6.353959999999992 }
{ "_id" : 1998, "value" : 5.262879999999994 }
{ "_id" : 1999, "value" : 5.646135458167332 }
{ "_id" : 2000, "value" : 6.030278884462145 }
{ "_id" : 2001, "value" : 5.02068548387097 }
{ "_id" : 2002, "value" : 4.61308 }
{ "_id" : 2003, "value" : 4.013879999999999 }
{ "_id" : 2004, "value" : 4.271320000000004 }
{ "_id" : 2005, "value" : 4.288880000000001 }
{ "_id" : 2006, "value" : 4.7949999999999955 }
{ "_id" : 2007, "value" : 4.634661354581674 }
{ "_id" : 2008, "value" : 3.6642629482071714 }
{ "_id" : 2009, "value" : 3.2641200000000037 }
has more
> 
~~~~

#### UFO Sightings

This will follow much of the same process as with the Treasury Yield
example with one extra step; we’ll need to add an entry into the build
file to compile this example. First, open the file for editing:

~~~~ {.code-java}
$ emacs project/MongoHadoopBuild.scala
~~~~

Next, add the following lines starting at line 72 in the build file:

~~~~ {.code-java}
...
  lazy val ufoExample = Project( id = "ufo-sightings",
                                base = file("examples/ufo_sightings"),
                                settings = exampleSettings ) dependsOn ( core )
...
~~~~

Now edit the UFO Sightings config file and update the `mongo.input.uri`
and `mongo.output.uri` properties:

~~~~ {.code-java}
$ emacs examples/ufo_sightings/src/main/resources/mongo-ufo_sightings.xml
...
  <property>
    <!-- If you are reading from mongo, the URI -->
    <name>mongo.input.uri</name>
    <value>mongodb://127.0.0.1/mongo_hadoop.ufo_sightings.in</value>
  </property>
  <property>
    <!-- If you are writing to mongo, the URI -->
    <name>mongo.output.uri</name>
    <value>mongodb://127.0.0.1/mongo_hadoop.ufo_sightings.out</value>
  </property>
...
~~~~

Next edit the main class for the MapReduce job in
`UfoSightingsXMLConfig.java` to use the configuration file:

~~~~ {.code-java}
$ emacs examples/ufo_sightings/src/main/java/com/mongodb/hadoop/examples/ufos/UfoSightingsXMLConfig.java
...
public class UfoSightingsXMLConfig extends MongoTool {

    static{
        // Load the XML config defined in hadoop-local.xml
        // Configuration.addDefaultResource( "hadoop-local.xml" );
        Configuration.addDefaultResource( "mongo-defaults.xml" );
        Configuration.addDefaultResource( "mongo-ufo_sightings.xml" );
    }

    public static void main( final String[] pArgs ) throws Exception{
        System.exit( ToolRunner.run( new UfoSightingsXMLConfig(), pArgs ) );
    }
}
...
~~~~

Now build the UFO Sightings example:

~~~~ {.code-java}
$ ./sbt ufo-sightings/package
~~~~

Once the example is built, execute the MapReduce job:

~~~~ {.code-java}
$ hadoop jar examples/ufo_sightings/target/ufo-sightings_cdh3u3-1.0.0.jar com.mongodb.hadoop.examples.UfoSightingsXMLConfig
~~~~

This MapReduce job will take just a bit longer than the Treasury Yield
example. Once it’s complete, check the output collection in MongoDB to
see that the job was successful:

~~~~ {.code-java}
$ mongo
MongoDB shell version: 2.0.5
connecting to: test
> use mongo_hadoop
switched to db mongo_hadoop
> db.ufo_sightings.out.find().count()
21850
~~~~

-   ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_square.gif)
    ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Introduction")
    [Introduction](http://www.mongodb.org/display/DOCS/Introduction)

-   ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_square.gif)
    ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Quickstart")
    [Quickstart](http://www.mongodb.org/display/DOCS/Quickstart)

-   [![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_plus.gif)](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#)
    ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Downloads")
    [Downloads](http://www.mongodb.org/display/DOCS/Downloads)
-   [![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_plus.gif)](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#)
    ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Drivers")
    [Drivers](http://www.mongodb.org/display/DOCS/Drivers)
-   [![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_plus.gif)](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#)
    ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Developer Zone")
    [Developer Zone](http://www.mongodb.org/display/DOCS/Developer+Zone)
-   [![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_plus.gif)](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#)
    ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Admin Zone")
    [Admin Zone](http://www.mongodb.org/display/DOCS/Admin+Zone)
-   ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_square.gif)
    ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Production Deployments")
    [Production
    Deployments](http://www.mongodb.org/display/DOCS/Production+Deployments)

-   [![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_plus.gif)](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#)
    ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Hosting Center")
    [Hosting Center](http://www.mongodb.org/display/DOCS/Hosting+Center)
-   [![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_plus.gif)](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#)
    ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Contributors")
    [Contributors](http://www.mongodb.org/display/DOCS/Contributors)
-   [![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_plus.gif)](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#)
    ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Community")
    [Community](http://www.mongodb.org/display/DOCS/Community)
-   [![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_minus.gif)](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#)
    ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "About")
    [About](http://www.mongodb.org/display/DOCS/About)
    -   ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_square.gif)
        ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Gotchas")
        [Gotchas](http://www.mongodb.org/display/DOCS/Gotchas)

    -   ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_square.gif)
        ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Philosophy")
        [Philosophy](http://www.mongodb.org/display/DOCS/Philosophy)

    -   [![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_minus.gif)](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#)
        ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Use Cases")
        [Use Cases](http://www.mongodb.org/display/DOCS/Use+Cases)
        -   ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_square.gif)
            ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Business Intelligence")
            [Business
            Intelligence](http://www.mongodb.org/display/DOCS/Business+Intelligence)

        -   ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_square.gif)
            ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Hadoop Quick Start")
            [Hadoop Quick
            Start](./Hadoop%20Quick%20Start%20-%20MongoDB_files/Hadoop%20Quick%20Start%20-%20MongoDB.html)

        -   ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_square.gif)
            ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Hadoop Scenarios")
            [Hadoop
            Scenarios](http://www.mongodb.org/display/DOCS/Hadoop+Scenarios)

        -   ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_square.gif)
            ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "How MongoDB is Used in Media and Publishing")
            [How MongoDB is Used in Media and
            Publishing](http://www.mongodb.org/display/DOCS/How+MongoDB+is+Used+in+Media+and+Publishing)

        -   ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_square.gif)
            ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Use Case - Session Objects")
            [Use Case - Session
            Objects](http://www.mongodb.org/display/DOCS/Use+Case+-+Session+Objects)

    -   ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_square.gif)
        ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "MongoDB-Based Applications")
        [MongoDB-Based
        Applications](http://www.mongodb.org/display/DOCS/MongoDB-Based+Applications)

    -   [![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_plus.gif)](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#)
        ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Events")
        [Events](http://www.mongodb.org/display/DOCS/Events)
    -   ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_square.gif)
        ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "MongoDB Archives and Articles")
        [MongoDB Archives and
        Articles](http://www.mongodb.org/display/DOCS/MongoDB+Archives+and+Articles)

    -   ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_square.gif)
        ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Benchmarks")
        [Benchmarks](http://www.mongodb.org/display/DOCS/Benchmarks)

    -   ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_square.gif)
        ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "FAQ")
        [FAQ](http://www.mongodb.org/display/DOCS/FAQ)

    -   [![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_plus.gif)](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#)
        ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Misc")
        [Misc](http://www.mongodb.org/display/DOCS/Misc)
    -   ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_square.gif)
        ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Licensing")
        [Licensing](http://www.mongodb.org/display/DOCS/Licensing)

-   ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_square.gif)
    ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "International Docs")
    [International
    Docs](http://www.mongodb.org/display/DOCS/International+Docs)

-   ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_square.gif)
    ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Books")
    [Books](http://www.mongodb.org/display/DOCS/Books)

-   ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_square.gif)
    ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Slides and Video")
    [Slides and
    Video](http://www.mongodb.org/display/DOCS/Slides+and+Video)

-   ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/tree_square.gif)
    ![](./Hadoop%20Quick%20Start%20-%20MongoDB_files/docs_16.gif "Alerts")
    [Alerts](http://www.mongodb.org/display/DOCS/Alerts)

[Follow @mongodb](http://www.twitter.com/mongodb)

Upcoming MongoDB Conferences

-   [Melbourne](http://www.10gen.com/events/mongodb-melbourne) - Nov 9
-   [Sydney](http://www.10gen.com/events/mongodb-sydney) - Nov 12
-   [Chicago](http://www.10gen.com/events/mongodb-chicago) - Nov 13
-   [Silicon Valley](http://www.10gen.com/events/mongosv) - Dec 4
-   [Tokyo](http://www.10gen.com/events/mongodb-tokyo) - Dec 12

* * * * *

-   Added by [Sandeep Parikh](http://www.mongodb.org/display/~crcsmnky),
    last edited by [Stephen
    Steneker](http://www.mongodb.org/display/~stephen.steneker@10gen.com)
    on Aug 12, 2012  ([view
    change](http://www.mongodb.org/pages/diffpages.action?pageId=40305037&originalId=42174492))
-   [show
    comment](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#)
    [hide
    comment](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#)

**Comment:** Correct path for TreasuryYieldXMLConfig.java\

Labels parameters

Labels
------

Enter labels to add to this page:

![Please wait](./Hadoop%20Quick%20Start%20-%20MongoDB_files/wait.gif) 

Looking for a label? Just start typing.

-   [Collapse
    All](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#)
    Collapsing… Expanding… [Expand
    All](http://www.mongodb.org/display/DOCS/Hadoop+Quick+Start#)

**[PLEASE POST QUESTIONS IN THE USER GROUPS
FORUM](http://groups.google.com/group/mongodb-user).** Post non-question
comments and helpful hints here.

Please enable JavaScript to view the \<a
href=“http://disqus.com/?ref\_noscript=mongodb”\>comments powered by
Disqus.\</a\>

\

Copyright © [10gen](http://www.10gen.com/), Inc. | Licensed under
[Creative Commons](http://creativecommons.org/licenses/by-nc-sa/3.0/). \
MongoDB®, Mongo®, and the leaf logo are registered trademarks of 10gen,
Inc.

-   Powered by [Atlassian
    Confluence](http://www.atlassian.com/software/confluence) 3.0.0\_01,
    the [Enterprise Wiki](http://www.atlassian.com/software/confluence).
-   Printed by Atlassian Confluence 3.0.0\_01, the Enterprise Wiki.
-   [Bug/feature
    request](http://jira.atlassian.com/secure/BrowseProject.jspa?id=10470)
    –
-   [Atlassian
    news](http://www.atlassian.com/about/connected.jsp?s_kwcid=Confluence-stayintouch)
    –
-   [Contact
    administrators](http://www.mongodb.org/administrators.action)
