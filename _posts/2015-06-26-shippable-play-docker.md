---
layout: post
title: Building a Play2 app into a Docker container using Shippable
---

Here I'm assuming you have basic knowledge of Docker and sbt.

If you have been taking steps to integrate Docker into your infrastructure, you may have come across (Shippable)[http://app.shippable.com].  They offer a hosted CI service at a decent price, (starting from free), but the main selling points for me were great support for both Docker and Scala.  Every build is run in a docker container, and they have prebuilt containers with the latest version of the Scala compiler pre-installed, all ready to run your build and tests.  Previously we were running Jenkins on an EC2 instance, but builds would take up to 20 minutes and maintenance was a drain on productivity.  Switching to Shippable compressed build times to less than 5 minutes and saved money into the bargain.  The build created aversioned zip file with the build output and upload it to a private S3 bucket, from where it was downloaded by the deploy server at provisioning time.  This setup worked great for a while, but we then decided to take it to the next level by dockerizing our Play2 app, hoping to get even faster deployments and better portability.  Shippable have recently launched a new service called 'Docker Build' which lets build a Docker container from your build artifacts, so this should have been a cynch, but there were a few gotchas along the way you'll want to know about if you are thinking of going down this route.

The go-to plugin for packing Play apps is the (native packager app)[https://github.com/sbt/sbt-native-packager], and it now has a Docker module.  There is also ()[], but native packager is officially supported, is under active development and seems well thought-out.

The native packager will actually write a `Dockerfile` for you, build a container from your app and push it to your local docker instance or a remote registry with virtually no configuration.  In `plugins.sbt`, add:

````scala
addSbtPlugin("com.typesafe.sbt" % "sbt-native-packager" % "1.0.1")
````

In your `Build.scala` (or `build.sbt`), you need to set the EntryPoint for the container (the program that will be run on container startup).  For a play app, it will be something like:

````scala
dockerEntrypoint := Seq(
    "bin/my-app-name",
    "-Dconfig.resource=my-application-name.conf",
    "-Dlogger.resource=my-app-logging-conf.xml")
````

Then fire off `sbt docker:publishLocal` and you're done - it works!  You should be able to fire up your app using `docker run` (use `docker images` to find the image name if you are not sure).

However, there's no such thing as a free lunch.  Having the Dockerfile generated automatically comes back to bite you.  For example, the default base image will include the entire OpenJDK, when only the JRE is needed, and you will need to set the package name if you want to push the image to a registry.  Further, Shippable expects that you check in your Dockerfile in the root of your repository, making automatic generation a no-no.  Therefore, we are going to write our own `Dockerfile` and configure Native Packager to use it.

````
FROM java:openjdk-7-jre
ADD ./buildoutput /opt
WORKDIR /opt/docker
RUN ["chown", "-R", "daemon:daemon", "."]
USER daemon
ENTRYPOINT [
    "bin/my-app-name",
    "-Dconfig.resource=my-app-name.conf",
    "-Dlogger.resource=my-app-logger.xml"
]
````

We use the jre base image, saving several hundred MB over the default `java` image, which contains the entire jdk. Also note the `/opt/docker` directory which is created by the Native Packager during `docker:stage`.

The second line is peculiar to Shippable.  Initially we assumed they would run `docker` over the entire contents of the build, but in actuality they do the following:
- Check out a fresh copy of your repo
- Copy the contents of `shippable/buildoutput` from your build into the fresh copy as `buildoutput`.
- Run docker in the fresh copy.

This means that you must copy all relevent build artifacts into `shippable/buildoutput` after the build, that you refer to all required files using `ADD ./buildoutput/...` and that you check in your Dockerfile to the root of your repo. Furthermore, if you get any of these wrong, debugging the issue can take a lot of time since you'll need to download, instantiate and attach to the resulting container, poking around in the file system to work out what went wrong.  To save you the heartache, here's an example `shippable.yml` which does the job:

````
build_image: shippableimages/ubuntu1404_scala
language: scala
scala:
    - 2.11.2
jdk:
    - openjdk7
before_script:
    - mkdir -p shippable/buildoutput
script: 
    - sbt test clean docker:stage
after_script:
    - cp -r target/docker/docker/stage/opt/* shippable/buildoutput/
cache: true
````

You can then set up the registry you want to push to in the Shippable web interface.

AFAIK, this is the only way to get your app into a docker image from Shippable.  A previous attempt to run docker inside Shippable was a waste of time (it's tempting, as then a simple `sbt docker:push` would have finished the job with no further ado). Apparently the Shippable containers don't have sufficient permission to run docker-within-docker.

After having worked through these issues so that you don't have to, I can say that everything now works very nicely, and deploy times have dropped from 30 minutes to 10.
