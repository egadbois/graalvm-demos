# Tiny Java Containers

This example shows how a simple Java application and a simple web
server can be compiled to produce very small Docker container images. 

The smallest container images contains just an executable.  But since there's
nothing in the container image except the executable, including no libc or other
shared libraries, an executable has to be fully statically linked with all
needed libraries and resources.

To support static linking libc, GraalVM Native Image supports using the
"lightweight, fast, simple, free" [musl](https://musl.libc.org/) libc
implementation.

You can watch a [Devoxx 2022](https://devoxx.be/) session that walks through
this example on YouTube. 

[![A 1.5MB Java Container
App](images/youtube.png)](https://youtu.be/6wYrAtngIVo)

## Prerequisites

You'll need a GraalVM release supporting **JDK 19** with Native Image installed.
This code was tested with GraalVM Enterprise Edition 22.3 for JDK 19. GraalVM
Community Edition 22.3 for JDK 19 works too, but Native Image generated
executables sizes will differ.  Download either with a single [GraalVM JDK
Downloader](https://github.com/graalvm/graalvm-jdk-downloader) command.

You'll also need Docker installed and running. It should work fine with
[podman](https://podman.io/) but it has not been tested.

NOTE: These instructions have only been tested on Linux amd64.

## Setup

Clone this Git repo and in your Linux shell type the following to download and
configure the `musl` toolchain.

![](images/keyboard.jpg) `$ ./setup-musl.sh`


## Hello World

With the `musl` toolchain installed, cd in to the `helloworld` folder.

![](images/keyboard.jpg) `cd helloworld`

Using the `build.sh` script, compile a simple single Java class Hello World
application with `javac`, compile the generated .class file into a fully
statically linked native Linux executable named `hello`, compress the executable
with [upx](https://upx.github.io/) to create the executable `hello.upx`, and
package the compress static `hello.upx` executable into a `scratch`
Docker container image:

![](images/keyboard.jpg) `$ ./build.sh`

You'll see two executables were built:

![](images/keyboard.jpg) `$ ls -lh hello*`

### The Executables

Running either of the `hello` executables you can see they are functionally
equivalent. They just print "Hello World". But there are a few points worth
noting:

1. The executable generated by GraalVM Native Image using the 
   `--static --libc=musl` options is a fully self-contained executable which can be
   confirmed by examining it with `ldd`:

   ![](images/keyboard.jpg) `$ ldd hello`

   should result in:

   ```shell
      not a dynamic executable
   ```

   This means that it does not rely on any libraries in the host operating system
   environment making it easier to package in a variety of Docker container images.

   Unfortunately `upx` compression renders `ldd` unable to list the shared
   libraries of an executable, but since we compressed the statically linked
   executable we can be confident it is also statically linked.

2. Both executables are the result of compiling a Java bytecode application into
   native machine code. The uncompressed executable is only 5.2MB!  There's no
   JVM, no jars, no JIT compiler and none of the overhead it imposes.  Both
   start extremely fast as there is minimal startup cost.
3. The `upx` compressed executable is about 60% smaller, 1.5MB vs. 5.2MB! With
   upx the application self-extracts quickly but does incur a cost of about
   100ms for decompression.  See this blog for a deep dive on [GraalVM Native Image and UPX](https://medium.com/graalvm/compressed-graalvm-native-images-4d233766a214).  

### Container Images

The size of the `scratch`-based container image is slightly more than the `hello.upx`
executable.  

```shell
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
hello               upx                 935e5e3549e6        1 second ago        1.51MB
```

This is a tiny container image and yet it contains a fully functional and
deployable (although fairly useless 😉) application.  The Dockerfile that
generated it simply copies the executable into the container image and sets the
executable as the `ENTRYPOINT`.  

A better way to build these images is with a multi-stage build, but to keep the
focus on the final result we'll build on a host machine and copy the binary into
the container image. E.g.,

```docker
FROM scratch
COPY hello.upx /
ENTRYPOINT ["/hello.upx"]
```

Running the container image is straight forward:

![](images/keyboard.jpg) `$ docker run --rm hello:upx`

```shell
Hello World
```

Amazingly, it works!

## A Simple Web Server

Containerizing Hello World is not that interesting so let's move on to something
you could actually deploy as a service. We'll take the [Simple Web
Server](https://blogs.oracle.com/javamagazine/post/java-18-simple-web-server)
introduced in JDK 18 and build a containerized executable that serves up web
pages.

How small can a containerized Java web server be? Would you believe a measly
5.5MB? Let's see.

Let's move from the `helloworld` folder over to the `jwebserver` folder. 

![](images/keyboard.jpg) `cd ../jwebserver`

There are a number of different GraalVM Native Image [linking
options](https://www.graalvm.org/22.0/reference-manual/native-image/StaticImages/)
that are suitable for different container images.

![](images/linkingoptions.png)

The `build-all.sh` script will generate a number of container images that
illustrate various linking and packaging options as well as a `jlink` generated
custome runtime image for comparison.

![](images/keyboard.jpg) `$ ./build-all.sh`

The various Dockerfiles simply copy the executable or `jlink` generated custom
runtime image folder into the container image along with an `index.html` file to
serve, and set the `ENTRYPOINT`.  E.g.,

```docker
FROM scratch
COPY jwebserver.static /
COPY index.html /web/index.html
ENTRYPOINT ["/jwebserver.static", "-b", "0.0.0.0", "-d", "/web"]
```

When complete you can see the sizes of the various versions:

![](images/keyboard.jpg) `$ docker images jwebserver`

```shell
REPOSITORY          TAG                            IMAGE ID            CREATED             SIZE
jwebserver          distroless-java-base.jlink     fae0bb62eca7        6 minutes ago       74.9MB
jwebserver          scratch.static-upx             676069a2a359        6 minutes ago       5.43MB
jwebserver          alpine.static                  14e748264a99        6 minutes ago       25MB
jwebserver          distroless-static.static       5591e1a2658a        6 minutes ago       21.8MB
jwebserver          scratch.static                 ef1ad68037ec        7 minutes ago       19.4MB
jwebserver          distroless-base.mostly         cc8612887001        7 minutes ago       39.7MB
jwebserver          distroless-java-base.dynamic   d2f802cf3def        8 minutes ago       58.7MB
```

Sorting by size, it's clear that the fully statically linked GraalVM Native
Image generated executable that's compressed and packaged on `scratch` is the
smallest at just 5.43MB, only 7% of the size of the `jlink` version running on
the JVM.

| Base Image           | App Version                        | Size (MB) |
| -------------------- | ---------------------------------- | --------- |
| Distroless Java Base | jlink                              |     74.90 |
| Distroless Java Base | native dynamic linked              |     58.70 |
| Distroless Base      | native *mostly* static linked      |     39.70 |
| Alpine               | native *fully* static              |     25.00 |
| Distroless Static    | native *fully* static              |     21.80 |
| Scratch              | native *fully* static              |     19.40 |
| Scratch              | *compressed* native *fully* static |      5.43 |

Running a container image is straight forward, just remember to map the ports, e.g.:

![](images/keyboard.jpg) `$ docker run --rm -p8000:8000 jwebserver:scratch.static`

or

![](images/keyboard.jpg) `$ docker run --rm -p8000:8000 jwebserver:scratch.static-upx`

Using `curl` or your favourite tool you can hit `http://localhost:8000` to fetch
the index.html file.

## Wrapping Up

There you have it.  Fully functional, albeit minimal, Java "microservice" compiled
into a native Linux executable and packaged into Distroless, Alpine, and
`scratch`-based container images thanks to GraalVM Native Image's support for
various linking options including fully static linking with the `musl` libc.

To learn more about linking options check out [Static and Mostly Static
Images](https://www.graalvm.org/22.0/reference-manual/native-image/StaticImages/)
in the GraalVM docs.
