---
out: Migrating-from-sbt-013x.html
---

  [Organizing-Build]: Organizing-Build.html

Migrating from sbt 0.13.x
-------------------------

### Migrating case class `.copy(...)`

Many of the case classes are replaced with pseudo case classes generated using Contraband. Migrate `.copy(foo = xxx)` to `withFoo(xxx)`.
Suppose you have `m: ModuleID`, and you're currently calling `m.copy(revision = "1.0.1")`. Here how you can migrate it:

```scala
m.withRevision("1.0.1")
```

### sbt version specific source directory

If you are cross building an sbt plugin, one escape hatch we have is sbt version specific source directory `src/main/scala-sbt-0.13` and `src/main/scala-sbt-1.0`. In there you can define an object named `PluginCompat` as follows:

```scala
package sbtfoo

import sbt._
import Keys._

object PluginCompat {
  type UpdateConfiguration = sbt.librarymanagement.UpdateConfiguration

  def subMissingOk(c: UpdateConfiguration, ok: Boolean): UpdateConfiguration =
    c.withMissingOk(ok)
}
```

Now `subMissingOk(...)` function can be implemented in sbt version specific way.

### Migrating to slash syntax

In sbt 0.13 keys were scoped with 2 different syntaxes: one for sbt's shell and one for in code.

* sbt 0.13 shell: `<project-id>/config:intask::key`
* sbt 0.13 code: `key in (<project-id>, Config, intask)`

Starting sbt 1.1.0, the syntax for scoping keys has been unified for both the shell and the build definitions to
the **slash syntax** as follows:

* `<project-id> / Config / intask / key`

Here are some examples:

```scala
lazy val root = (project in file("."))
  .settings(
    name := "hello",
    version in ThisBuild := "1.0.0-SNAPSHOT",
    scalacOptions in Compile += "-Xlint",
    scalacOptions in (Compile, console) --= Seq("-Ywarn-unused", "-Ywarn-unused-import"),
    fork in Test := true
  )
```

They are now written as:

```scala
lazy val root = (project in file("."))
  .settings(
    name := "hello",
    ThisBuild / version := "1.0.0-SNAPSHOT",
    Compile / scalacOptions += "-Xlint",
    Compile / console / scalacOptions --= Seq("-Ywarn-unused", "-Ywarn-unused-import"),
    Test / fork := true
  )
```

And now the same syntax in sbt's shell:

```
sbt:hello> name
[info] hello
sbt:hello> ThisBuild / version
[info] 1.0.0-SNAPSHOT
sbt:hello> show Compile / scalacOptions
[info] * -Xlint
sbt:hello> show Compile / console / scalacOptions
[info] * -Xlint
sbt:hello> Test / fork
[info] true
```

To aid migration a [Scalafix](https://scalacenter.github.io/scalafix/) rewrite rule was created. After setting
up Scalafix (either locally to a project or user-wide) you should be able to migrate your build with:

```scala
> sbtfix github:sbt/sbt/v1_1
```

and migrate your sbt plugin code with:
```scala
> scalafix github:sbt/sbt/v1_1
```

### Migrating from sbt 0.12 style

Before sbt 0.13 (sbt 0.9 to 0.12) it was very common to see in builds the usage of three aspects of sbt:

* the key dependency operators: `<<=`, `<+=`, `<++=`
* the tuple enrichments (apply and map) for TaskKey's and SettingKey's (eg. `(foo, bar) map { (f, b) => ... }`)
* the use of `Build` trait in `project/Build.scala`

The release of sbt 0.13 (which was over 3 years ago!) introduced the `.value` DSL which allowed for much
easier to read and write code, effectively making the first two aspects redundant and they were removed from the official
documentation.

Similarly, sbt 0.13's introduction of multi-project `build.sbt` made the `Build` trait redundant.
In addition, the auto plugin feature that's now standard in sbt 0.13 enabled automatic sorting of plugin settings
and auto import feature, but it made `Build.scala` more difficult to maintain.

As they are removed in sbt 1.0.0, and here we'll help guide you to how to migrate your code.

#### Migrating sbt 0.12 style operators

With simple expressions such as:

```scala
a <<= aTaskDef
b <+= bTaskDef
c <++= cTaskDefs
```

it is sufficient to replace them with the equivalent:

```scala
a := aTaskDef.value
b += bTaskDef.value
c ++= cTaskDefs.value
```

#### Migrating from the tuple enrichments

As mentioned above, there are two tuple enrichments `.apply` and `.map`. The difference used to be for whether
you're defining a setting for a `SettingKey` or a `TaskKey`, you use `.apply` for the former and `.map` for the
latter:

```scala
val sett1 = settingKey[String]("SettingKey 1")
val sett2 = settingKey[String]("SettingKey 2")
val sett3 = settingKey[String]("SettingKey 3")

val task1 = taskKey[String]("TaskKey 1")
val task2 = taskKey[String]("TaskKey 2")
val task3 = taskKey[String]("TaskKey 3")
val task4 = taskKey[String]("TaskKey 4")

sett1 := "s1"
sett2 := "s2"
sett3 <<= (sett1, sett2)(_ + _)

task1 := { println("t1"); "t1" }
task2 := { println("t2"); "t2" }
task3 <<= (task1, task2) map { (t1, t2) => println(t1 + t2); t1 + t2 }
task4 <<= (sett1, sett2) map { (s1, s2) => println(s1 + s2); s1 + s2 }
```

(Remember you can define tasks in terms of settings, but not the other way round)

With the `.value` DSL you don't have to know or remember if your key is a `SettingKey` or a `TaskKey`:

```scala
sett1 := "s1"
sett2 := "s2"
sett3 := sett1.value + sett2.value

task1 := { println("t1"); "t1" }
task2 := { println("t2"); "t2" }
task3 := { println(task1.value + task2.value); task1.value + task2.value }
task4 := { println(sett1.value + sett2.value); sett1.value + sett2.value }
```

#### Migrating when using `.dependsOn`, `.triggeredBy` or `.runBefore`

When instead calling `.dependsOn`, instead of:

```scala
a <<= a dependsOn b
```

define it as:

```scala
a := (a dependsOn b).value
```

**Note**: You'll need to use the `<<=` operator with `.triggeredBy` and `.runBefore` in sbt 0.13.13 and
earlier due to issue [#1444](https://github.com/sbt/sbt/issues/1444).

#### Migrating when you need to set `Task`s

For keys such as `sourceGenerators` and `resourceGenerators` which use sbt's Task type:

```scala
val sourceGenerators =
  settingKey[Seq[Task[Seq[File]]]]("List of tasks that generate sources")
val resourceGenerators =
  settingKey[Seq[Task[Seq[File]]]]("List of tasks that generate resources")
```

Where you previous would define things as:

```scala
sourceGenerators in Compile <+= buildInfo
```

for sbt 1, you define them as:

```scala
Compile / sourceGenerators += buildInfo
```

or in general,

```scala
Compile / sourceGenerators += Def.task { List(file1, file2) }
```

#### Migrating with `InputKey`

When using `InputKey` instead of:

```scala
run <<= docsRunSetting
```

when migrating you mustn't use `.value` but `.evaluated`:

```scala
run := docsRunSetting.evaluated
```

### Migrating from the Build trait

With `Build` trait based build such as:

```scala
import sbt._
import Keys._
import xyz.XyzPlugin.autoImport._

object HelloBuild extends Build {
  val shared = Defaults.defaultSettings ++ xyz.XyzPlugin.projectSettings ++ Seq(
    organization := "com.example",
    version      := "0.1.0",
    scalaVersion := "2.12.1")

  lazy val hello =
    Project("Hello", file("."),
      settings = shared ++ Seq(
        xyzSkipWrite := true)
    ).aggregate(core)

  lazy val core =
    Project("hello-core", file("core"),
      settings = shared ++ Seq(
        description := "Core interfaces",
        libraryDependencies ++= scalaXml.value)
    )

  def scalaXml = Def.setting {
    scalaBinaryVersion.value match {
      case "2.10" => Nil
      case _      => ("org.scala-lang.modules" %% "scala-xml" % "1.0.6") :: Nil
    }
  }
}
```

You can migrate to `build.sbt`:

```scala
val shared = Seq(
  organization := "com.example",
  version      := "0.1.0",
  scalaVersion := "2.12.1"
)

lazy val helloRoot = (project in file("."))
  .aggregate(core)
  .enablePlugins(XyzPlugin)
  .settings(
    shared,
    name := "Hello",
    xyzSkipWrite := true
  )

lazy val core = (project in file("core"))
  .enablePlugins(XyzPlugin)
  .settings(
    shared,
    name := "hello-core",
    description := "Core interfaces",
    libraryDependencies ++= scalaXml.value
  )

def scalaXml = Def.setting {
  scalaBinaryVersion.value match {
    case "2.10" => Nil
    case _      => ("org.scala-lang.modules" %% "scala-xml" % "1.0.6") :: Nil
  }
}
```

1. Rename `project/Build.scala` to `build.sbt`.
2. Remove import statements `import sbt._`, `import Keys._`, and any auto imports.
3. Move all of the inner definitions (like `shared`, `helloRoot`, etc) out of the `object HelloBuild`, and remove `HelloBuild`.
4. Change `Project(...)` to `(project in file("x"))` style, and call its `settings(...)` method to pass in the settings. This is so the auto plugins can reorder their setting sequence based on the plugin dependencies. `name` setting should be set to keep the old names.
5. Remove `Defaults.defaultSettings` out of `shared` since these settings are already set by the built-in auto plugins, also remove `xyz.XyzPlugin.projectSettings` out of `shared` and call `enablePlugins(XyzPlugin)` instead.

**Note**: `Build` traits is deprecated, but you can still use `project/*.scala` file to organize your build and/or define ad-hoc plugins. See [Organizing the build][Organizing-Build].

