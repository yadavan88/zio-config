---
id: configdescriptorusage_index
title:  "Using ConfigDescriptor for Read, Write, Document and Report"
---

## Using Config Descriptor

Given a single [ConfigDescriptor](../configdescriptor/index.md) we can use it to:

1. Get Readers that can read config from various sources
2. Get Writer that can write config back
3. Document the config
4. Create report on the config


## Reader from Config Descriptor

You should be familiar with reading config from various sources, given a  config descriptor.

```scala mdoc:silent
import zio.IO
import zio.config._, ConfigDescriptor._, PropertyTree._

```

```scala mdoc:silent

case class MyConfig(ldap: String, port: Int, dburl: String)

```

To not divert our focus on handling Either (only for explanation purpose), we will use 
the below syntax troughout the code


To perform any action using zio-config, we need a configuration description.
Let's define a simple one.


```scala mdoc:silent

val myConfig =
  (string("LDAP") zip int("PORT") zip string("DB_URL")).to[MyConfig]

read(myConfig from ConfigSource.fromMap(Map()))

```

## Writer from config descriptor

Let's use `read` and get the result. Using the same result, we will write the config back to the source!

```scala mdoc:silent
import zio.Runtime

  case class Database(url: String, port: Int)
  case class AwsConfig(c1: Database, c2: Database, c3: String)

  val database =
    (string("connection") zip int("port")).to[Database]

  val map =
    Map(
      "south.connection" -> "abc.com",
      "south.port" -> "8111",
      "east.connection" -> "xyz.com",
      "east.port" -> "8888",
      "appName" -> "myApp"
    )

  val appConfig =
    (((nested("south") { database } ?? "South details" zip
      nested("east") { database } ?? "East details" zip
      string("appName")).to[AwsConfig]) ?? "asdf"
    ) from ConfigSource.fromMap(map, keyDelimiter = Some('.'))

  val awsConfigResult =
    zio.Runtime.default.unsafeRun(read(appConfig))

   // yields AwsConfig(Database(abc.com, 8111), Database(xyz.com, 8888), myApp)

  
```

#### Writing the config back to property tree

```scala mdoc:silent

import zio.config.PropertyTree._

val written: Either[String, PropertyTree[String, String]] = 
  write(appConfig, awsConfigResult)


```

yield 

```scala

  Record(
    Map(
      "south"   -> Record(Map("connection" -> Leaf("abc.com"), "port" -> Leaf("8111"))),
      "east"    -> Record(Map("connection" -> Leaf("xyz.com"), "port" -> Leaf("8888"))),
      "appName" -> Leaf("myApp")
    )
  ) 

```

#### Writing the config back to a Map

```scala mdoc:silent

 // To yield the input map that was fed in, call `flattenString` !!
 written.map(_.flattenString())

 // yields
     Map(
      "south.connection" -> "abc.com",
      "south.port" -> "8111",
      "east.connection" -> "xyz.com",
      "east.port" -> "8888",
      "appName" -> "myApp"
    )

```

#### Writing the config back to a Typesafe Hocon

```scala mdoc:silent

import zio.config.typesafe._

written.map(_.toHocon)

// yields
// SimpleConfigObject({"appName":"myApp","east":{"connection":"xyz.com","port":"8888"},"south":{"connection":"abc.com","port":"8111"}})

```

#### Writing the config back to a Typesafe Hocon String

Once we get `SimpleConfigObject` (i.e, from `toHocon`), rendering them is straight forward, as typesafe-config library
provides us with an exhaustive combinations of rendering options.

However, we thought we will provide a few helper functions which is a simple delegation to typesafe functionalities.

```scala mdoc:silent

import zio.config.typesafe._

written.map(_.toHoconString)
  /**
   *  yieling :
   *  {{{
   *
   *    appName=myApp
   *    east {
   *       connection="xyz.com"
   *       port="8888"
   *    }
   *   south {
   *     connection="abc.com"
   *     port="8111"
   *   }
   *
   *  }}}
   */

```

#### Writing the config back to JSON

Once we get `SimpleConfigObject` (i.e, from `toHocon`), rendering them is straight forward, as typesafe-config library
provides us with an exhaustive combinations of rendering options.

However, we thought we will provide a few helper functions which is a simple delegation to typesafe functionalities.

```scala mdoc:silent

written.map(_.toJson)

  /**
   *  yieling :
   *  {{{
   *
   *    "appName" : "myApp"
   *    "east" : {
   *       "connection" : "xyz.com"
   *       "port" : "8888"
   *    }
   *   "south" : {
   *     "connection" : "abc.com"
   *     "port" : "8111"
   *   }
   *
   *  }}}
   */
```

## Generating a random Config using zio.config.gen

```scala mdoc:silent

import zio.config.derivation.name
import zio.config.magnolia._, zio.config.gen._

object RandomConfigGenerationSimpleExample extends App {
  sealed trait Region

  @name("ap-southeast-2")
  case object ApSouthEast2 extends Region

  @name("us-east")
  case object UsEast extends Region

  case class UsernameRegion(username: String, region: Region)

  println(generateConfigJson(descriptor[UsernameRegion]).unsafeRunChunk)

  // yields for example

  // Chunk(
  //   {
  //    "region" : "ap-southeast-2",
  //     "username" : "eU2KlfATwYZ5s0Y"
  //   }
  // )
}


```

Refer to RandomConfigGenerationComplexExample.scala for more complex scenarios,
and know how zio.config.gen can be helpful for users.

## Document the config


To generate the documentation of the config, call `generateDocs`. 


```scala mdoc:silent
 import zio.config.ConfigDocs

 val generatedDocs = generateDocs(appConfig)

 // as markdown 
  val markdown =
     generatedDocs.toTable.toGithubFlavouredMarkdown

 // produces the following markdown

  /*
    |FieldName      |Format          |Description               |Sources |
    |---            |---             |---                       |---     |
    |appName        |primitive       |value of type string, asdf|constant|
    |[east](#east)  |[all-of](#east) |                          |        |
    |[south](#south)|[all-of](#south)|                          |        |
    
    ### east
    
    |FieldName |Format   |Description                             |Sources |
    |---       |---      |---                                     |---     |
    |port      |primitive|value of type int, East details, asdf   |constant|
    |connection|primitive|value of type string, East details, asdf|constant|
    
    ### south
    
    |FieldName |Format   |Description                              |Sources |
    |---       |---      |---                                      |---     |
    |port      |primitive|value of type int, South details, asdf   |constant|
    |connection|primitive|value of type string, South details, asdf|constant|
   */  
```

In the above `markdown` is a standard markdown format string, rendered as:


   |FieldName             |Format                 |Description               |Sources |
   |---                   |---                    |---                       |---     |
   |appName               |primitive              |value of type string, asdf|constant|
   |[east](#root.east_1)  |[all-of](#root.east_1) |                          |        |
   |[south](#root.south_0)|[all-of](#root.south_0)|                          |        |
   
   ### root.east_1
   
   |FieldName |Format   |Description                             |Sources |
   |---       |---      |---                                     |---     |
   |port      |primitive|value of type int, East details, asdf   |constant|
   |connection|primitive|value of type string, East details, asdf|constant|
   
   ### root.south_0
   
   |FieldName |Format   |Description                              |Sources |
   |---       |---      |---                                      |---     |
   |port      |primitive|value of type int, South details, asdf   |constant|
   |connection|primitive|value of type string, South details, asdf|constant|
   

## Report on the config

Calling `generateDocs` can give some documentation (man page).
But most often, we need these docs to act as a report that holds the value of the actual config parameter
along with the rest of the details. 


```scala mdoc:silent

 generateReport(appConfig, AwsConfig(Database("abc.com", 8111), Database("xyz.com", 8888), "myApp"))

// yields a report

```

Pretty print will be coming soon!
