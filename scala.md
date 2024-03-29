## Adding Shutdown Hooks for JVM Applications
Refer: https://www.baeldung.com/jvm-shutdown-hooks

The JVM allows registering functions to run before it completes its shutdown. These functions are usually a good place for releasing resources or other similar house-keeping tasks. In JVM terminology, these functions are called shutdown hooks.

Shutdown hooks are basically initialized but unstarted threads. When the JVM begins its shutdown process, it will start all registered hooks in an unspecified order. After running all hooks, the JVM will halt.

```scala
// Scala
Runtime.getRuntime.addShutdownHook(new Thread() {
    override def run(): Unit = {
      println(""In the middle of a shutdown"")
      // Close resources gracefully
    }
})

// Java
Thread printingHook = new Thread(() -> System.out.println("In the middle of a shutdown"));
Runtime.getRuntime().addShutdownHook(printingHook);
```

The JVM is responsible for starting hook threads. Therefore, if the given hook has been already started, Java will throw an exception:

```java
Thread longRunningHook = new Thread(() -> {
    try {
        Thread.sleep(300);
    } catch (InterruptedException ignored) {}
});
longRunningHook.start();

assertThatThrownBy(() -> Runtime.getRuntime().addShutdownHook(longRunningHook))
  .isInstanceOf(IllegalArgumentException.class)
  .hasMessage("Hook already running");
```

Obviously, we also can't register a hook multiple times:
```
Thread unfortunateHook = new Thread(() -> {});
Runtime.getRuntime().addShutdownHook(unfortunateHook);

assertThatThrownBy(() -> Runtime.getRuntime().addShutdownHook(unfortunateHook))
  .isInstanceOf(IllegalArgumentException.class)
  .hasMessage("Hook previously registered");
```

The JVM can be shut down in two different ways:
1. A controlled process
2. An abrupt manner

A controlled process shuts down the JVM when either:
- The last non-daemon thread terminates. For example, when the main thread exits, the JVM starts its shutdown process
- Sending an interrupt signal from the OS. For instance, by pressing `Ctrl + C` or logging off the OS
- Calling `System.exit()` from Java code

While we all strive for graceful shutdowns, sometimes the JVM may shut down in an abrupt and unexpected manner. The JVM shuts down abruptly when:
- Sending a kill signal from the OS. For example, by issuing a `kill -9 <jvm_pid>`
- Calling `Runtime.getRuntime().halt()` from Java code
- The host OS dies unexpectedly, For example, in a power failure or OS panic

The JVM runs shutdown hooks only in case of normal terminations. So, when an external force kills the JVM process abruptly, the JVM won't get a chance to execute shutdown hooks. Additionally, halting the JVM from Java code will also have the same effect:

```scala
Thread haltedHook = new Thread(() -> System.out.println("Halted abruptly"));
Runtime.getRuntime().addShutdownHook(haltedHook);
        
Runtime.getRuntime().halt(129);
```

The halt method forcibly terminates the currently running JVM. Therefore, registered shutdown hooks won't get a chance to execute.

## Change Current Directory
`cd` is not actually a program. It's a shell-internal that tells the shell to the chdir system call. 

Java either doesn't have a function to make the entire jvm to change its cwd (current working directory). One option is executing `sh -c '...'`, changing the directory within a forked process, like so:
```scala
import sys.process._
val cmd = "whatever you wanted to run"
s"sh -c 'cd ../..; $cmd'".!
```

Still better would be to use the `scala.sys.process.Process` factory that take both a command and a cwd:
```scala
scala.sys.process.Process("your command here", new java.io.File("/some/dir"))
```

**Refer**: https://stackoverflow.com/a/34969373/1879109

## Class Constructors
```scala
class MyException(message: String) extends Exception(message) {

  def this(message: String, cause: Throwable) {
    this(message)
    initCause(cause)
  }

  def this(cause: Throwable) {
    this(Option(cause).map(_.toString).orNull, cause)
  }

  def this() {
    this(null: String)
  }
}
```

## Convert Java Collections to Scala
Starting Scala 2.13, package `scala.jdk.CollectionConverters` replaces packages `scala.collection.JavaConverters/JavaConversions._`:

```scala
import scala.jdk.CollectionConverters._

val javaList: java.util.List[String] = java.util.Arrays.asList("one","two","three")

javaList.asScala
// collection.mutable.Buffer[String] = Buffer("one", "two", "three")

javaList.asScala.toSet
// collection.immutable.Set[String] = Set("one", "two", "three")
```

## Convert a List into a Tuple

There isn't any way in standard Scala library to convert a list into a tuple as:

`List(1, 2, 3)` to `(1, 2, 3)`

<br/>
We can however use **Shapeless** library to create a HList first, which then can be converted into a tuple.

``` scala
scala> val hl = 1:: 2:: 3:: HNil
hl: shapeless.::[Int,shapeless.::[Int,shapeless.::[Int,shapeless.HNil]]] = 1 :: 2 :: 3 :: HNil

scala> hl.tupled
res6: (Int, Int, Int) = (1,2,3)
```

## Convert List of lists into Tuple of lists

There isn't any way in standard Scala library to convert a list of lists into tuple of lists as in:

`List(List(1, 2, 3), List(4, 5, 6), List(7, 8, 9))` to `(List(1, 2, 3), List(4, 5, 6), List(7, 8, 9))`

<br/>
We can however use **Shapeless** library to create a HList of lists first, which then can be converted into a tuple.

``` scala
scala> import shapeless._
import shapeless._

scala> import HList._
import HList._

scala> val hlist = List(1, 2, 3) :: List(4, 5, 6) :: List(7, 8, 9) :: HNil
hlist: shapeless.::[List[Int],shapeless.::[List[Int],shapeless.::[List[Int],shapeless.HNil]]] = List(1, 2, 3) :: List(4, 5, 6) :: List(7, 8, 9) :: HNil

scala> println(hlist)
List(1, 2, 3) :: List(4, 5, 6) :: List(7, 8, 9) :: HNil

scala> hlist.tupled
res2: (List[Int], List[Int], List[Int]) = (List(1, 2, 3),List(4, 5, 6),List(7, 8, 9))
```

## Convert List of tuples into a List of lists

Tuples can't directly be converted to lists. What we instead can do is to get an **Iterator** for that tuple, which then can be converted into a list.

``` scala
scala> val c = List(('a,'b,'c), ('b,'c,'d), ('c,'d,'e), ('d,'e,'f), ('e,'f,'a), ('f,'a,'b))
c: List[(Symbol, Symbol, Symbol)] = List(('a,'b,'c), ('b,'c,'d), ('c,'d,'e), ('d,'e,'f), ('e,'f,'a), ('f,'a,'b))

scala> c map {_.productIterator toList}
warning: there was one feature warning; re-run with -feature for details
res4: List[List[Any]] = List(List('a, 'b, 'c), List('b, 'c, 'd), List('c, 'd, 'e), List('d, 'e, 'f), List('e, 'f, 'a), List('f, 'a, 'b))
```

## Convert String to LocalDateTime
``` scala
scala> LocalDateTime.of(LocalDate.parse("17-02-2020", 
  DateTimeFormatter.ofPattern("dd-MM-yyyy")), LocalDateTime.now().toLocalTime())
res14: java.time.LocalDateTime = 2020-02-17T09:54:44.503

scala> LocalDateTime.of(LocalDate.parse("17-02-2020", 
  DateTimeFormatter.ofPattern("dd-MM-yyyy")), LocalDateTime.MIN.toLocalTime())
res15: java.time.LocalDateTime = 2020-02-17T00:00

LocalDateTime.parse("Jan 15, 2019 20:12", DateTimeFormatter.ofPattern("MMM dd, yyyy HH:mm")) 
//2019-01-15T20:12

LocalDateTime.parse("09/25/2017 12:55 PM", DateTimeFormatter.ofPattern("MM/dd/yyyy hh:mm a")) 
// 2017-09-25T12:55

LocalDateTime.parse("02-August-1989 11:40:12.450", DateTimeFormatter.ofPattern("dd-MMMM-yyyy HH:mm:ss.SSS")) 
// 1989-08-02T11:40:12.450
```

**Refer**: https://www.baeldung.com/java-date-to-localdate-and-localdatetime

## Count the number of occurrences in a list
Refer: https://stackoverflow.com/questions/11448685/scala-how-can-i-count-the-number-of-occurrences-in-a-list

```scala
val s = Seq("apple", "oranges", "apple", "banana", "apple", "oranges", "oranges")
s.groupBy(l => l).map(t => (t._1, t._2.length))

// Or
s.groupBy(identity).mapValues(_.size)

// Results
Map(banana -> 1, oranges -> 3, apple -> 3)
```

## Create/Update configs in Scala at runtime
Cases where you have to load default `application.conf` but also override or add runtime configurations.

```scala
package com.iamsmkr.configs

import com.iamsmkr.configs.RuntimeConfig.ConfigBuilder
import com.typesafe.config.{Config, ConfigFactory, ConfigValueFactory}

class RuntimeConfig private(config: Config) {
  def getConfig: Config = config
}

object RuntimeConfig {
  case class ConfigBuilder() {
    var tempConf: Config =
      ConfigFactory
        .defaultOverrides()
        .withFallback(ConfigFactory.defaultApplication())

    def addConfig(key: String, value: Any): ConfigBuilder = {
      tempConf = tempConf
        .withValue(
          key,
          ConfigValueFactory.fromAnyRef(value)
        )
      this
    }

    def build(): RuntimeConfig = new RuntimeConfig(tempConf.resolve())
  }
}

object TestConfig {

  val r: RuntimeConfig =
    ConfigBuilder()
    .addConfig("raphtory.deploy.address", "127.0.0.1")
    .addConfig("raphtory.deploy.port", 1736)
    .build()

  r.getConfig
}
```

## Find duplicates in a list
Refer: https://stackoverflow.com/questions/24729544/how-to-find-duplicates-in-a-list

```scala
val dup = List(1,1,1,2,3,4,5,5,6,100,101,101,102)
dup.groupBy(identity).collect { case (x, List(_,_,_*)) => x }

// Or
dup.diff(dup.distinct).distinct
```

## Find files with a given extension
```scala
if("/tmp".toDirectory.files.map(_.path).exists(name => name matches """.*\.gz"""))
  Seq("/bin/sh", "-c", s"gunzip $dataDir/*.gz").!
```

**Refer**: https://stackoverflow.com/a/48271851/1879109

## How to pick up a free port number on localhost
Do not bind to a specific port. Instead, bind to port 0. 

Port 0 carries special significance in network programming, particularly in the Unix OS when it comes to socket programming where the port is used to request system-allocated, dynamic ports. Port 0 is a wildcard port that tells the system to find a suitable port number.

Refer: https://www.lifewire.com/port-0-in-tcp-and-udp-818145

## Load Configurations
`ConfigFactory` loads `application.conf` present under `src/main/resources` or `src/test/resources`.
```scala
val config = ConfigFactory.load()
val dataDir: String = config.getString("raphtory.spout.file.local.sourceDirectory")
```

If you wish to load any configuration file named other than `application.conf`, you can pass the same as argument to the `load` method of `ConfigFactory`.
```scala
val config = ConfigFactory.load("reference.conf")
val dataDir: String = config.getString("raphtory.spout.file.local.sourceDirectory")
```

**Refer**: https://github.com/lightbend/config#standard-behavior

## Managed resources
The resource passed to `using` as first parameter will be closed after the function passed as second is done with it's execution. The resource however is expected to implement the `close` function that takes care of closing the resource.

```scala
def using[A <: {def close() : Unit}, B](resource: A)(f: A => B): B = {
  try {
    f(resource)
  } finally {
    resource.close()
  }
}
  
// usage
val content = using(Source.fromFile("/tmp/myfile.txt"))(_.mkString)
```

## Map.map vs Map.mapValues
**Refer:** http://blog.bruchez.name/2013/02/mapmap-vs-mapmapvalues.html

## Output Redirection
Bash like output redirection can be implemented in Scala using implicit classes like so.

```scala
trait Pipeline {

  implicit class toPiped[V](value: V) {
    def |>[R](f: V => R) = f(value)
  }

}
 
 // usage
 def uuid: String = java.util.UUID.randomUUID.toString
 
def createUniqueTmpDir(): String = {
  val dir = new File(s"/tmp/$uuid")
  dir.mkdir()

  dir.getAbsolutePath
}

def writeZipToTmpFile(tmpDir: String)(implicit zipAsByteArray: Array[Byte]): File = {
  val zip = new File(tmpDir + "/1")

  using(new FileOutputStream(zip)) { os =>
    os.write(zipAsByteArray)
    zip
  }
}

def unzip(zip: File): String = {
  ZipUtil.explode(zip)

  s"${zip.getParent}/1"
}

implicit val zipAsByteArray: Array[Byte] = ??? 

createUniqueTmpDir |> writeZipToTmpFile |> unzip

// dependencies
"org.zeroturnaround" % "zt-zip" % "1.13"
```

## Marshalling/Unmarshalling Java LocalDateTime using spray-json
Marshalling/Unmarshalling of Java LocalDateTime is not natively supported in spray-json as of now. It could be achieved by manually writing reader and writer like so:
``` scala
implicit object LocalDateTimeJsonFormat extends RootJsonFormat[LocalDateTime] {
    override def write(dt: LocalDateTime): JsValue =
      JsString(dt.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME))

    override def read(json: JsValue): LocalDateTime = json match {
      case JsString(s) => LocalDateTime.parse(s, DateTimeFormatter.ISO_LOCAL_DATE_TIME)
      case _ => throw DeserializationException("Decode local datetime failed")
    }
 }
```

**Refer**
1. https://www.programcreek.com/scala/java.time.LocalDateTime
2. https://github.com/spray/spray-json/issues/128#issuecomment-258712233

## Marshalling/Unmarshalling Scala Enumerations using spray-json
[Refer PR](https://github.com/spray/spray-json/pull/336)

Marshalling/Unmarshalling of Scala Enumerations is not natively supported in spray-json as of now. It could be achieved by manually writing reader and writer like so:
```scala
import spray.json._

class EnumJsonFormat[T <: scala.Enumeration](enu: T) extends RootJsonFormat[T#Value] {
  override def write(obj: T#Value): JsValue = JsString(obj.toString)
  override def read(json: JsValue): T#Value = {
    json match {
      case JsString(txt) => enu.withName(txt)
      case somethingElse => throw DeserializationException(s"Expected a value from enum $enu instead of $somethingElse")
    }
  }
}

object Fruits extends Enumeration {
  type Fruit = Value
  val APPLE, BANANA, MANGO = Value
}

it("should be possible to serialize/deserialize enum") {
  implicit val fruitFormat: EnumJsonFormat[Fruits.type] = new EnumJsonFormat(Fruits)
  Fruits.APPLE.toJson should be(JsString("APPLE"))
  JsString("BANANA").convertTo[Fruits.Fruit] should be(Fruits.BANANA)
}
```

## Marshalling/Unmarshalling generic ADT using spray-json
Suppose you need to reply to a rest api request in a predefined response format, as shown below. 
```json
{
    "data": {
        "id": "123",
        "fullName": "Shivam Kapoor"
    }
}

{ "error": "No employee found" }
```

We can define ADTs to abstract these responses, as shown below. Clearly, successfully responses need to be of type generic so as to capture all sorts of responses.
```scala
case class SuccessHttpResponse[T](data: T, msg: Option[String] = None)
case class FailureHttpResponse(error: String)
```

This woud require json serializer/deserializer to be defined as below:
```scala
implicit def successHttpResponseFormat[T: JsonFormat]: RootJsonFormat[SuccessHttpResponse[T]] = jsonFormat2(SuccessHttpResponse.apply[T])
implicit val failureHttpResponseFormat: RootJsonFormat[FailureHttpResponse] = jsonFormat1(FailureHttpResponse)
```

## Read configs in Scala using `pureconfig`

This reason you would use a scala library instead of **Lightbend's** already popular [configuration library](https://github.com/lightbend/config) because it is written purely in Java and when you try to fetch complex formats instead of simply fetching configs by `String` or `Int` it returns *Java Collections* which makes it difficult to work with in Scala especially if the configs are deeply nested. 

Basically, it can't cast configs to *Case Classes*.

``` scala
case class TtlPerDatabase(cass: Option[Int], solr: Option[Int], vertica: Option[Int])

import java.nio.file.Paths
import pureconfig.generic.auto._
import scala.util.{Failure, Success, Try}

trait ReadConfigs {
    def confPath

    def fetchTtls() = {
        // pureconfig.loadConfig[Map[String, TtlPerDatabase]](ConfigFactory.parseString("""ttl: { "vce/vce/pod": {cass:80, solr:90, vertica:10}, "ibm/ibm/pod": {cass:21, solr:7, vertica:3} }"""), "scalar.ttl")

        val ttlConf = pureconfig.loadConfig[Map[String, TtlPerDatabase]](Paths.get(confPath), "scalar.ttl")
        Try {
          ttlConf match {
            case Left(configReaderFailures) =>
              sys.error(s"Encountered the following errors reading ttl configurations per mps: ${configReaderFailures.toList.mkString("\n")}")
            case Right(config) =>
              config
          }
        }
      }
     
  }
  
  // usage
  fetchTtls() match {
    case Success(ttls) => println(ttls)
    case Failure(e) => e.printStackTrace()
  }
```

## Read from a file

```scala
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

import scala.io.Source

object MyFileReader {
  private def using[A <: {def close() : Unit}, B](resource: A)(f: A => B): B =
    try {
      f(resource)
    } finally {
      resource.close()
    }

  def printData(log: String) = Future {
    using(Source.fromFile(log)) { source =>
      source.getLines.toList.foreach(x => println(x))
    }
  }
}

```

## Read resource file abs path 
Reading the absolute path of a file present in the resource dir serves no purpose because you cannot read the entries within a `jar` like it was a plain old `File`. The absolute path looks something like this: `file:/example.jar!/file.txt`.

Rather than trying to address the resource as a `File` just ask the `ClassLoader` to return an `InputStream` for the resource instead via `getResourceAsStream`:
```scala
try (InputStream in = getClass().getResourceAsStream("/file.txt");
    BufferedReader reader = new BufferedReader(new InputStreamReader(in))) {
    // Use resource
}
```

Refer: https://stackoverflow.com/questions/20389255/reading-a-resource-file-from-within-jar

## Reload your configuration files reactively 

1. Poll config file for changes
Conventional way would be to poll checksum of the configuation file every configured period of time, check if the checksum has changed and based on the results reload the file.

2. Using `better-files-akka` library in Scala

https://github.com/pathikrit/better-files

``` scala
import akka.actor.{ActorRef, ActorSystem}
import better.files.File
import better.files.FileWatcher._
import java.nio.file.{StandardWatchEventKinds => EventType}

trait ConfWatcher {
  def confDir: String
  implicit def actorSystem: ActorSystem

  private val confPath = confDir + "application.conf"
  private val appConfFile = File(confPath)
  private var appConfLastModified = appConfFile.lastModifiedTime

  val watcher: ActorRef = appConfFile.newWatcher(recursive = false)

  watcher ! on(EventType.ENTRY_MODIFY) { file =>
    if (appConfLastModified.compareTo(file.lastModifiedTime) < 0) {
      // TODO
      appConfLastModified = file.lastModifiedTime
    }
  }

}
```

**Note:** This snippet of code also address multiple events [issue](https://github.com/pathikrit/better-files/issues/313).

## String format
``` scala
"%s %s, age %d".format(firstName, lastName, age)
```

## Using Scala Reflection to create instance of a Case Class with default values
Refer: https://stackoverflow.com/questions/16939511/instantiating-a-case-class-with-default-args-via-reflection

```scala
case class Weekday(i: Int = 0, str: String = "Sunday")

def newDefault[A](implicit t: reflect.ClassTag[A]): A = {
  import reflect.runtime.{universe => ru, currentMirror => cm}

  val clazz  = cm.classSymbol(t.runtimeClass)
  val mod    = clazz.companion.asModule
  val im     = cm.reflect(cm.reflectModule(mod).instance)
  val ts     = im.symbol.typeSignature
  val mApply = ts.member(ru.TermName("apply")).asMethod
  val syms   = mApply.paramLists.flatten
  val args   = syms.zipWithIndex.map { case (p, i) =>
    val mDef = ts.member(ru.TermName(s"apply$$default$$${i+1}")).asMethod
    im.reflectMethod(mDef)()
  }
  im.reflectMethod(mApply)(args: _*).asInstanceOf[A]
}

val f = newDefault[Weekday]
println(f)
```

## Using Scala Reflection to invoke members of companion object
The following example is in comparison to creating instances as shown in previous listing. 
```scala
object TestDefault extends App {

  case class XYZ(str: String = "Shivam")
  object XYZ { private val default: XYZ = XYZ() }
  case class ABC(int: Int = 99)
  object ABC { private val default: ABC = ABC() }

  def newDefault[A](implicit t: reflect.ClassTag[A]): A = {
    import reflect.runtime.{universe => ru}
    import reflect.runtime.{currentMirror => cm}

    val clazz  = cm.classSymbol(t.runtimeClass)
    val mod    = clazz.companion.asModule
    val im     = cm.reflect(cm.reflectModule(mod).instance)
    val ts     = im.symbol.typeSignature
    val mApply = ts.member(ru.TermName("apply")).asMethod
    val syms   = mApply.paramLists.flatten
    val args   = syms.zipWithIndex.map {
      case (p, i) =>
        val mDef = ts.member(ru.TermName(s"apply$$default$$${i + 1}")).asMethod
        im.reflectMethod(mDef)()
    }
    im.reflectMethod(mApply)(args: _*).asInstanceOf[A]
  }

  for (i <- 0 to 1000000000)
    newDefault[XYZ]

//  println(s"newDefault XYZ = ${newDefault[XYZ]}")
//  println(s"newDefault ABC = ${newDefault[ABC]}")

  def newDefault2[A](implicit t: reflect.ClassTag[A]): A = {
    import reflect.runtime.{currentMirror => cm}

    val clazz = cm.classSymbol(t.runtimeClass)
    val mod   = clazz.companion.asModule
    val im    = cm.reflect(cm.reflectModule(mod).instance)
    val ts    = im.symbol.typeSignature

//    val methodMap = ts.members
//      .filter(_.isMethod)
//      .map(d => {
//        d.name.toString -> d.asMethod
//      })
//      .toMap

    val defaultMember = ts.members.filter(_.isMethod).filter(d => d.name.toString == "default").head.asMethod

    val result = im.reflectMethod(defaultMember).apply()
//    val result = im.reflectMethod(methodMap("default")).apply()
    result.asInstanceOf[A]
  }

  for (i <- 0 to 1000000000)
    newDefault2[XYZ]
}
```

**Note**: Doesn't seem to make much difference with reflections! 

newDefault
![Screenshot-2022-09-19-at-3-32-26-pm.png](https://i.postimg.cc/nc7qNCSG/Screenshot-2022-09-19-at-3-32-26-pm.png)


newDefault2
![Screenshot-2022-09-19-at-3-33-22-pm.png](https://i.postimg.cc/nzyC11xb/Screenshot-2022-09-19-at-3-33-22-pm.png)

Memoization does infact makes some difference!

**Refer**: https://stackoverflow.com/questions/73775296/avoiding-garbage-while-creating-objects-using-scala-runtime-reflection

```scala
  val cache = new ConcurrentHashMap[universe.Type, XYZ]()

  def newDefault2[A](implicit t: reflect.ClassTag[A]): A = {
    import reflect.runtime.{currentMirror => cm}

    val clazz = cm.classSymbol(t.runtimeClass)
    val mod   = clazz.companion.asModule
    val im    = cm.reflect(cm.reflectModule(mod).instance)
    val ts    = im.symbol.typeSignature

    if (!cache.contains(ts)) {
      val default = ts.members.filter(_.isMethod).filter(d => d.name.toString == "default").head.asMethod
      cache.put(ts, im.reflectMethod(default).apply().asInstanceOf[XYZ])
    }

    cache.get(ts).asInstanceOf[A]
  }

  for (i <- 0 to 1000000000)
    newDefault2[XYZ]
```

![Screenshot-2022-09-19-at-9-53-07-pm.png](https://i.postimg.cc/HsnKyqbB/Screenshot-2022-09-19-at-9-53-07-pm.png)


## Using wildcards with scala.sys.process._ in Scala
Refer: https://stackoverflow.com/questions/71132425/using-wildcards-with-scala-sys-process-in-scala

## "View" in Scala

View produces a lazy collection, so that calls to e.g. `filter` do not evaluate every element of the collection. *Elements are only evaluated once they are explicitly accessed.*

``` scala
scala> (1 to 1000000000).filter(_ % 2 == 0).take(10).toList
java.lang.OutOfMemoryError: GC overhead limit exceeded
```
Here Scala tries to create a collection with 1000000000 elements to then access the first 10. But with view:

``` scala
scala> (1 to 1000000000).view.filter(_ % 2 == 0).take(10).toList
res2: List[Int] = List(2, 4, 6, 8, 10, 12, 14, 16, 18, 20)
```

**Refer:** [In Scala, what does “view” do?](http://stackoverflow.com/questions/6799648/in-scala-what-does-view-do)

## Wait for all futures to complete
A `Future` produced by `Future.sequence` completes when either:
- all the futures have completed successfully, or
- one of the futures has failed

The second point is what's happening in your case, and it makes sense to complete as soon as one of the wrapped `Future` has failed, because the wrapping `Future` can only hold a single `Throwable` in the failure case. There's no point in waiting for the other futures because the result will be the same failure.

However, it's perfectly reasonable to want to gather all the results, failed or not.

``` scala
def waitAll[T](futures: Seq[Future[T]]): Future[Seq[Try[T]]] = {
  def lift[X](futures: Seq[Future[X]]): Seq[Future[Try[X]]] =
    futures.map(_.map(Success(_)).recover { case t => Failure(t) })

  // Having neutralized exception completions through the lifting, .sequence can now be used
  // Refer: https://stackoverflow.com/a/29344937/1879109
  Future.sequence(lift(futures))
}
```

## Ways to deal with Option in Scala

``` scala
val r: Option[String] = Some("one")
//    val r: Option[String] = None

// First method: pattern matching
val res1 = r match {
  case Some(o) => "Result is " + o
  case None => "Not Found"
}
println(res1)

// Second method: built-in map combinator
val res2 = r.map(x => "Result is " + x).getOrElse("Not Found")
println(res2)

// Third method: built-in fold combinator
val res3 = r.fold("Not Found") {
  "Result is " + _
}
println(res3)
```

## Write to a file
```scala
import java.nio.file.{Paths, Files}
import java.nio.charset.StandardCharsets

Files.write(Paths.get("file.txt"), "file contents".getBytes(StandardCharsets.UTF_8))
```

Using `sys.process._`
```scala
import sys.process._
"echo hello world" #> new java.io.File("/tmp/example.txt") !
```

Refer: https://stackoverflow.com/questions/6879427/scala-write-string-to-file-in-one-statement

## Zip elements from multiple lists

We are aware of zipping two lists as:

``` scala
scala> List(1, 2, 3) zip List(4, 5, 6)
res3: List[(Int, Int)] = List((1,4), (2,5), (3,6))
```

<br/>
It is however also possible to zip more number of lists as:

``` scala
scala> val c = (List('a, 'b, 'c, 'd, 'e, 'f), List('b, 'c, 'd, 'e, 'f, 'a), List('c, 'd, 'e, 'f, 'a, 'b)).zipped.toList
c: List[(Symbol, Symbol, Symbol)] = List(('a,'b,'c), ('b,'c,'d), ('c,'d,'e), ('d,'e,'f), ('e,'f,'a), ('f,'a,'b))
```
