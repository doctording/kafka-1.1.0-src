Kafka概念和源码学习

## 基础概念

### topic

* Topic是Kafka数据写入操作的基本单元，可以指定副本
* 一个Topic包含一个或多个Partition，建Topic的时候可以手动指定Partition个数，个数与服务器个数相当
每条消息属于且仅属于一个Topic
* Producer发布数据时，必须指定将该消息发布到哪个Topic
* Consumer订阅消息时，也必须指定订阅哪个Topic的信息

### Partition

* 每个Partition在物理上对应一个文件夹，该文件夹下存储这个Partition的所有消息和索引文件
* 一个Partition中的数据是有序、不可变的，使用偏移量(offset)唯一标识一条数据，是一个long类型的数据

#### 为什么要分区的一些原因

1. 分区类似hadoop分布式文件系统，也类似于spark中RDD以partition方式将数据分块存储，一是分块存储提高数据概率性安全，比如分布式的文件块，如果某台机器宕机数据丢失，也是丢失一部分，不会出现整个文件全军覆没。另外通过partition级别的冗余存储（replication.factor）来保证partition级别的安全性。

2. 每个partition可以被认为是一个无限长度的数组，新数据顺序追加进这个数组。物理上，每个partition对应于一个文件夹。一个broker上可以存放多个partition。这样，producer可以将数据发送给多个broker上的多个partition，consumer也可以并行从多个broker上的不同partition上读数据，实现了水平扩展

3. 并行读写：磁盘写入速度就是kafka处理速度的极限，处理不过来就要加机器。每台机器持有不同的partition。Producer产生10个新消息，可以交给10个broker上的partition一起写；Consumer消费10个新消息，可以从10个broker上面一起读。

4. The partitions in the log serve several purposes. First, they allow the log to scale beyond a size that will fit on a single server. Each individual partition must fit on the servers that host it, but a topic may have many partitions so it can handle an arbitrary amount of data. Second they act as the unit of parallelism—more on that in a bit.The partitions of the log are distributed over the servers in the Kafka cluster with each server handling data and requests for a share of the partitions. Each partition is replicated across a configurable number of servers for fault tolerance.


### kafka的ack机制

`request.required.acks`有三个值 0 1 -1(all)

* 0: 生产者不会等待broker的ack，这个延迟最低但是存储的保证最弱当server挂掉的时候就会丢数据
* 1: 服务端会等待ack值，leader副本确认接收到消息后发送ack但是如果leader挂掉后他不确保是否复制完成新leader也会导致数据丢失
* -1: 同样在1的基础上，服务端会等所有的follower的副本受到数据后才会受到leader发出的ack，这样数据不会丢失


### 消费消息，offset概念

Kafka会为每一个Consumer Group保留一些metadata信息——当前消费的消息的position，也即offset。这个offset由Consumer控制。正常情况下Consumer会在消费完一条消息后递增该offset。当然，Consumer也可将offset设成一个较小的值，重新消费一些消息。因为offet由Consumer控制，所以Kafka broker是无状态的，它不需要标记哪些消息被哪些消费过，也不需要通过broker去保证同一个Consumer Group只有一个Consumer能消费某一条消息，因此也就不需要锁机制，这也为Kafka的高吞吐率提供了有力保障。

---

Apache Kafka
=================
See our [web site](http://kafka.apache.org) for details on the project.

You need to have [Gradle](http://www.gradle.org/installation) and [Java](http://www.oracle.com/technetwork/java/javase/downloads/index.html) installed.

Kafka requires Gradle 3.0 or higher.

Java 7 should be used for building in order to support both Java 7 and Java 8 at runtime.

### First bootstrap and download the wrapper ###
    cd kafka_source_dir
    gradle

Now everything else will work.

### Build a jar and run it ###
    ./gradlew jar

Follow instructions in http://kafka.apache.org/documentation.html#quickstart

### Build source jar ###
    ./gradlew srcJar

### Build aggregated javadoc ###
    ./gradlew aggregatedJavadoc

### Build javadoc and scaladoc ###
    ./gradlew javadoc
    ./gradlew javadocJar # builds a javadoc jar for each module
    ./gradlew scaladoc
    ./gradlew scaladocJar # builds a scaladoc jar for each module
    ./gradlew docsJar # builds both (if applicable) javadoc and scaladoc jars for each module

### Run unit/integration tests ###
    ./gradlew test # runs both unit and integration tests
    ./gradlew unitTest
    ./gradlew integrationTest
    
### Force re-running tests without code change ###
    ./gradlew cleanTest test
    ./gradlew cleanTest unitTest
    ./gradlew cleanTest integrationTest

### Running a particular unit/integration test ###
    ./gradlew -Dtest.single=RequestResponseSerializationTest core:test

### Running a particular test method within a unit/integration test ###
    ./gradlew core:test --tests kafka.api.ProducerFailureHandlingTest.testCannotSendToInternalTopic
    ./gradlew clients:test --tests org.apache.kafka.clients.MetadataTest.testMetadataUpdateWaitTime

### Running a particular unit/integration test with log4j output ###
Change the log4j setting in either `clients/src/test/resources/log4j.properties` or `core/src/test/resources/log4j.properties`

    ./gradlew -i -Dtest.single=RequestResponseSerializationTest core:test

### Generating test coverage reports ###
Generate coverage reports for the whole project:

    ./gradlew reportCoverage

Generate coverage for a single module, i.e.: 

    ./gradlew clients:reportCoverage
    
### Building a binary release gzipped tar ball ###
    ./gradlew clean
    ./gradlew releaseTarGz

The above command will fail if you haven't set up the signing key. To bypass signing the artifact, you can run:

    ./gradlew releaseTarGz -x signArchives

The release file can be found inside `./core/build/distributions/`.

### Cleaning the build ###
    ./gradlew clean

### Running a task on a particular version of Scala (either 2.11.x or 2.12.x) ###
*Note that if building the jars with a version other than 2.11.12, you need to set the `SCALA_VERSION` variable or change it in `bin/kafka-run-class.sh` to run the quick start.*

You can pass either the major version (eg 2.11) or the full version (eg 2.11.12):

    ./gradlew -PscalaVersion=2.11 jar
    ./gradlew -PscalaVersion=2.11 test
    ./gradlew -PscalaVersion=2.11 releaseTarGz

Scala 2.12.x requires Java 8.

### Running a task for a specific project ###
This is for `core`, `examples` and `clients`

    ./gradlew core:jar
    ./gradlew core:test

### Listing all gradle tasks ###
    ./gradlew tasks

### Building IDE project ####
*Note that this is not strictly necessary (IntelliJ IDEA has good built-in support for Gradle projects, for example).*

    ./gradlew eclipse
    ./gradlew idea

The `eclipse` task has been configured to use `${project_dir}/build_eclipse` as Eclipse's build directory. Eclipse's default
build directory (`${project_dir}/bin`) clashes with Kafka's scripts directory and we don't use Gradle's build directory
to avoid known issues with this configuration.

### Building the jar for all scala versions and for all projects ###
    ./gradlew jarAll

### Running unit/integration tests for all scala versions and for all projects ###
    ./gradlew testAll

### Building a binary release gzipped tar ball for all scala versions ###
    ./gradlew releaseTarGzAll

### Publishing the jar for all version of Scala and for all projects to maven ###
    ./gradlew uploadArchivesAll

Please note for this to work you should create/update `${GRADLE_USER_HOME}/gradle.properties` (typically, `~/.gradle/gradle.properties`) and assign the following variables

    mavenUrl=
    mavenUsername=
    mavenPassword=
    signing.keyId=
    signing.password=
    signing.secretKeyRingFile=

### Publishing the streams quickstart archetype artifact to maven ###
For the Streams archetype project, one cannot use gradle to upload to maven; instead the `mvn deploy` command needs to be called at the quickstart folder:

    cd streams/quickstart
    mvn deploy

Please note for this to work you should create/update user maven settings (typically, `${USER_HOME}/.m2/settings.xml`) to assign the following variables

    <settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                           https://maven.apache.org/xsd/settings-1.0.0.xsd">
    ...                           
    <servers>
       ...
       <server>
          <id>apache.snapshots.https</id>
          <username>${maven_username}</username>
          <password>${maven_password}</password>
       </server>
       <server>
          <id>apache.releases.https</id>
          <username>${maven_username}</username>
          <password>${maven_password}</password>
        </server>
        ...
     </servers>
     ...


### Installing the jars to the local Maven repository ###
    ./gradlew installAll

### Building the test jar ###
    ./gradlew testJar

### Determining how transitive dependencies are added ###
    ./gradlew core:dependencies --configuration runtime

### Determining if any dependencies could be updated ###
    ./gradlew dependencyUpdates

### Running code quality checks ###
There are two code quality analysis tools that we regularly run, findbugs and checkstyle.

#### Checkstyle ####
Checkstyle enforces a consistent coding style in Kafka.
You can run checkstyle using:

    ./gradlew checkstyleMain checkstyleTest

The checkstyle warnings will be found in `reports/checkstyle/reports/main.html` and `reports/checkstyle/reports/test.html` files in the
subproject build directories. They are also are printed to the console. The build will fail if Checkstyle fails.

#### Findbugs ####
Findbugs uses static analysis to look for bugs in the code.
You can run findbugs using:

    ./gradlew findbugsMain findbugsTest -x test

The findbugs warnings will be found in `reports/findbugs/main.html` and `reports/findbugs/test.html` files in the subproject build
directories.  Use -PxmlFindBugsReport=true to generate an XML report instead of an HTML one.

### Common build options ###

The following options should be set with a `-P` switch, for example `./gradlew -PmaxParallelForks=1 test`.

* `commitId`: sets the build commit ID as .git/HEAD might not be correct if there are local commits added for build purposes.
* `mavenUrl`: sets the URL of the maven deployment repository (`file://path/to/repo` can be used to point to a local repository).
* `maxParallelForks`: limits the maximum number of processes for each task.
* `showStandardStreams`: shows standard out and standard error of the test JVM(s) on the console.
* `skipSigning`: skips signing of artifacts.
* `testLoggingEvents`: unit test events to be logged, separated by comma. For example `./gradlew -PtestLoggingEvents=started,passed,skipped,failed test`.
* `xmlFindBugsReport`: enable XML reports for findBugs. This also disables HTML reports as only one can be enabled at a time.

### Running in Vagrant ###

See [vagrant/README.md](vagrant/README.md).

### Contribution ###

Apache Kafka is interested in building the community; we would welcome any thoughts or [patches](https://issues.apache.org/jira/browse/KAFKA). You can reach us [on the Apache mailing lists](http://kafka.apache.org/contact.html).

To contribute follow the instructions here:
 * http://kafka.apache.org/contributing.html
