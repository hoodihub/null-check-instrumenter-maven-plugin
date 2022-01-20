## __Null-Check Instrumenter maven plugin__

This plugin is based on osundblad's IntelliJ IDEA annotations instrumenter plugin (1.1.0), which itself was based on
Vlad Rassokhin's original intellij-annotations-instrumenter-maven-plugin.

This plugin extends osundblad's plugin by an option to let the instrumented code just log an error message, instead of
throwing an exception. This allows you to migrate more safely to a null-checked code base, as in the first step you can
go live with just the logging, and then enable the 'real deal' (exception throwing) once you are more certain that it
won't generate a lot of exceptions.

## What does it do

This plugin inserts null checks on non-null annotated parameters and/or method return values in you byte code, and also
inserts code to either throw an exception or log an error (based on configuration), when a non-null annotated value is
actually null.

## Usage

Add the following plugin definition to your build:

```xml

<build>
  <plugins>
    <plugin>
      <plugin>
        <groupId>net.hoodihub</groupId>
        <artifactId>null-check-instrumenter-maven-plugin</artifactId>
        <version>1.0.0</version>
        <executions>
          <execution>
            <id>instrument</id>
            <goals>
              <goal>instrument</goal>
              <goal>tests-instrument</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugin>
    ...
  </plugins>
</build>
```

and start adding `@NotNull`/`@Nullable` annotations to your code.

## Use other and/or multiple annotations

By default, only the annotation `org.jetbrains.annotations.NotNull` is supported. However you can define a set of
annotation types as follows:

```xml

<build>
  <plugins>
    <plugin>
      ...
      <configuration>
        <notNull>
          <param>org.jetbrains.annotations.NotNull</param>
          <param>javax.validation.constraints.NotNull</param>
        </notNull>
      </configuration>
    </plugin>
  </plugins>
</build>
```

This will instrument both jetbrains and javax annotations.

Note that configuration will replace the default annotations, so `org.jetbrains.annotations.NotNull` will no longer be
included by default, if you don't add it explicitly.

## Implicit NotNull instrumentation

If you don't like to have `@NotNull` on 99.99% of your parameters and methods, turn on the implicit instrumentation:

```xml

<build>
  <plugins>
    <plugin>
      ...
      <configuration>
        <implicit>true</implicit>
        <nullable>
          <param>org.jetbrains.annotations.Nullable</param>
        </nullable>
      </configuration>
    </plugin>
  </plugins>
</build>
```

This will instrument all parameters and return values with NotNull unless annotated with Nullable. I.e.

```java
public String implicit(String a,String b){
        if(a.equals(b)){
        return null;
        }
        return a+b;
        }
```

will trigger instrumentation if either the a or b parameter is null, or if a equals b (since it is not allowed to return
null). To allow nulls you would have to annotate the parameters/return value like this:

```java
@Nullable
public String implicit(@Nullable String a,@Nullable String b){
        if(a.equals(b)){
        return null;
        }
        return a+b;
        }
```

which would trigger instrumentation if a is null, return null if a equals b, and otherwise append the Strings (or a +
null if b is null).

**Note** that when using implicit you need to specify the Nullable annotations (not the NotNull).

## Use Objects.requireNonNull instead of explicitly throwing exceptions

With the configuration parameter 'useRequireNonNull' = true, the plugin will instrument Objects.requireNonNull calls,
instead of explicitly throwing an exception.

Note: This setting has a higher priority than logErrorInsteadOfThrowingException. If both settings are enabled, then
useRequireNonNull will win.

## Log errors instead of throwing exceptions

If you want the instrumented code to just log an error message instead of throwing an exception, you can use a
configuration like this:

```xml

<build>
  <plugins>
    <plugin>
      ...
      <configuration>
        <logErrorInsteadOfThrowingException>true</logErrorInsteadOfThrowingException>
        <loggerName>MyLogger</loggerName>
      </configuration>
    </plugin>
  </plugins>
</build>
```

This will generate code like:

```java
org.slf4j.LoggerFactory.getLogger("MyLogger").error(...);
```

ATTENTION: As you can see from this code, you need to make sure that your application always has `Slf4j` available in
the classpath, otherwise the instrumentation will fail.

The log message will contain details on what parameter/return value was null.

## Turn off Instrumentation

The property `se.eris.notnull.instrument=true/false` turns on/off the instrumentation. This may seem like a stupid
feature but it is really useful when you have multiple maven profiles and only one of them, eg Sonar/Findbugs, should
build without instrumentation since it messes up the statistics (branch coverage, complexity, etc).

`mvn clean install -Dse.eris.notnull.instrument=false`

## Exclusion

To ease migration to implicit it is now possible to exclude certain class files from instrumentation. This is still a
bit experimental the exclusion rules might change (depending on feedback).

There are three patterns

* __.__  matching package boundary
* __\*__  matching anything except package boundaries
* __\*\*__  matching anything (including package boundaries)
* __.\*\*__  matching any number of package levels

Example:

```xml

<configuration>
  <implicit>true</implicit>
  <excludes>
    <classes>**.wsdl.**</classes>
    <classes>com.*.*Spec</classes>
  </excludes>
</configuration>
```

Would exclude all files which have wsdl in any package part and classes with names ending in Spec under
com.&lt;package&gt; for example com.a.UnitSpec but not com.a.b.UnitSpec or com.UnitSpec.

