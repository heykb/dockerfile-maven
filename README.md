# Dockerfile Maven Support [![Build Status](https://travis-ci.com/spotify/dockerfile-maven.svg?token=1Nm1N8cG5T5xoHi9Uo1X&branch=master)](https://travis-ci.com/spotify/dockerfile-maven) [![Maven Central](https://img.shields.io/maven-central/v/com.spotify/dockerfile-maven.svg)](https://search.maven.org/#search%7Cga%7C1%7Cg%3A%22com.spotify%22%20dockerfile-maven*) [![License](https://img.shields.io/github/license/spotify/dockerfile-maven.svg)](LICENSE.txt)

This is a Maven plugin which tries to seamlessly integrate Docker with
Maven.

The design goals are:

  - Don't try to do anything fancy.  `Dockerfile`s are how you build
    Docker projects; that's what this plugin uses.  They are
    mandatory.
  - Make the Docker build process integrate with the Maven build
    process.  If you bind the default phases, when you type `mvn
    package`, you get a Docker image.  When you type `mvn deploy`,
    your image gets pushed.
  - Make the goals remember what you are doing.  You can type `mvn
    dockerfile:build` and later `mvn dockerfile:tag` and later `mvn
    dockerfile:push` without problems.  This also eliminates the need
    for something like `mvn dockerfile:build -DalsoPush`; instead you
    can just say `mvn dockerfile:build dockerfile:push`.
  - Integrate with the Maven build reactor.  You can depend on the
    Docker image of one project in another project, and Maven will
    build the projects in the correct order.  This is useful when you
    want to run integration tests involving multiple services.

This project adheres to the [Open Code of Conduct][code-of-conduct].
By participating, you are expected to honor this code.

[code-of-conduct]: https://github.com/spotify/code-of-conduct/blob/master/code-of-conduct.md

# Set-up

This plugin requires Java 7 or later, and Apache Maven 3 or later.  To
run the integration tests or to use the plugin in practice, a working
Docker set-up is needed.

# Example

For more examples, see the [integration test](./plugin/src/it) directory.

In particular, the [advanced](./plugin/src/it/advanced) test showcases a
full service consisting of two micro-services that are integration
tested using `helios-testing`.

This configures the actual plugin to build your image with `mvn
package` and push it with `mvn deploy`.  Of course you can also say
`mvn dockerfile:build` explicitly.

```xml
<plugin>
  <groupId>com.spotify</groupId>
  <artifactId>dockerfile-maven-plugin</artifactId>
  <version>${version}</version>
  <executions>
    <execution>
      <id>default</id>
      <goals>
        <goal>build</goal>
        <goal>push</goal>
      </goals>
    </execution>
  </executions>
  <configuration>
    <repository>spotify/foobar</repository>
    <tag>${project.version}</tag>
  </configuration>
</plugin>
```

A corresponding `Dockerfile` could look like:

```
FROM dockerfile/java:oracle-java8
MAINTAINER David Flemström <dflemstr@spotify.com>

ENTRYPOINT ["/usr/bin/java", "-jar", "/usr/share/myservice/myservice.jar"]

# Add Maven dependencies (not shaded into the artifact; Docker-cached)
ADD target/lib           /usr/share/myservice/lib
# Add the service itself
ADD target/myservice.jar /usr/share/myservice/myservice.jar
```

# What does it give me?

There are many advantages to using this plugin for your builds.

## Faster build times

This plugin lets you leverage Docker cache more consistently, vastly
speeding up your builds by letting you cache Maven dependencies in
your image.  It also encourages avoiding the `maven-shade-plugin`,
which also greatly speeds up builds.

## Consistent build lifecycle

You no longer have to say something like:

    mvn package
    mvn docker:build
    mvn verify
    mvn docker:push
    mvn deploy

Instead, it is simply enough to say:

    mvn deploy

With the basic configuration, this will make sure that the image is
built and pushed at the correct times.

## Depend on Docker images of other services

You can depend on the Docker information of another project, because
this plugin attaches project metadata when it builds Docker images.
Simply add this information to any project:

```xml
<dependency>
  <groupId>com.spotify</groupId>
  <artifactId>foobar</artifactId>
  <version>1.0-SNAPSHOT</version>
  <type>docker-info</type>
</dependency>
```

Now, you can read information about the Docker image of the project
that you depended on:

```java
String imageName = getResource("META-INF/docker/com.spotify/foobar/image-name");
```

This is great for an integration test where you want the latest
version of another project's Docker image.

Note that you have to register a Maven extension in your POM (or a
parent POM) in order for the `docker-info` type to be supported:

```xml
<build>
  <extensions>
    <extension>
      <groupId>com.spotify</groupId>
      <artifactId>dockerfile-maven-extension</artifactId>
      <version>${version}</version>
    </extension>
  </extensions>
</build>
```

## Use other Docker tools that rely on Dockerfiles

Your project(s) look like so:

```
a/
  Dockerfile
  pom.xml
b/
  Dockerfile
  pom.xml
```

You can now use these projects with Fig or docker-compose or some
other system that works with Dockerfiles.  For example, a
`docker-compose.yml` might look like:

```yaml
service-a:
  build: a/
  ports:
  - '80'

service-b:
  build: b/
  links:
  - service-a
```

Now, `docker-compose up` and `docker-compose build` will work as
expected.