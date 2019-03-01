# Security scanning as part of a CI/CD pipeline
* Initial tip #1: Make your life easier by using Docker [multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/). This makes it easy to create readable `Dockerfile` that result in small images.
* Initial tip #2: Use BuildKit. Enable it locally and on your build server! The guide can be found [here](https://docs.docker.com/develop/develop-images/build_enhancements/#to-enable-buildkit-builds).
  * Builds in parallel.
  * Builds only required stages, as opposed to simply building all stages, starting with the first one _listed_ in the `Dockerfile`.
  * By default, only failing steps are shown and the rest are summarized, with the time spent next to each one. Use `--progress=plain` to get all output from all steps performed.

## Advantages of using Docker for all CI steps
* Everybody on your team can now get the "works on my machine" effect on the build server!

## Good Docker habits
We list some good Docker habits that should be learned by every developer. 

### Always use the latest version of the base image, no matter what it is
Chances are your base image gets patched regularly (even if it is tied to a version of Java, for example). To make sure that you have the latest patches, always add `--pull`, which will check for newer versions of the base image.

<pre>
docker build <b>--pull</b> .
</pre>

You are encouraged to pick a base image you are using and view its history. Chances are you will be surprised at the number of updates that have been applied to it very recently. Use `--pull`! 

### For your own base images, rebuild them periodically
If you are using a self-made base image, make sure to rebuild it periodically and push it to the registry! A nightly CI/CD job is great for this. Of course, use `--pull` while building these.

### Experiment with different base images
There are a lot of different base images for everything. Experiment with them. Find the right version and tag that suits the state of your project. Try to find an image that includes what you need so that you can simplify your `Dockerfile`, or make your own.

# Things to remember for various project types
Different projects have different needs and concerns. We try to address some of these for Java, Python and npm projects in particular.

## All projects
There are a lot of things that should be done for most, if not all, projects using Docker.

### Choose the right trade-off
First and foremost, you want (in no particular order):
* A simple and easy-to-maintain Dockerfile
* A secure image
* An image that takes a reasonable amount of time
* An image that utilizes build cache well
* A small image

In general, this means:
* For intermediate build images, use an image that has "everything" so that few or no additional installation commands are required. Do not be afraid to have many `RUN` lines in your `Dockerfile`, as this will make it more readable. Also, adding a line will make Docker utilize the cache for everything that came before it, but modifying a line of, say, a list of packages that needs to be installed will cause Docker to install *all* the packages again.
* For the final image, have as few layers as possible. Add and remove one-time-needed things in the same layer. Sacrifice some readability, particularly if it is for the sake of eliminating vulnerabilities. Utilize multi-stage builds fully; use a minimal base image and copy what you need from previous stages. Make sure no dependencies required only for development or testing are present.

### Install (and copy) only what you need, purge what you can
Unless it causes inflexibility in the `Dockerfile` (and/or you know you *really* need everything), try to avoid installing every recommended dependency. For example, running `apt-get install gcc` on `debian:latest` installs `manpages`, which you likely do not need. For `apt`, add `--no-install-recommends` to _only_ install the packages you request. This might not matter if you are using the image only for building (other than increased build time, of course). Some other commands that can be useful:

* `apt-get remove`
* `apt-get purge`
* `apt-get autoremove`
* `apt-get autoclean`
* `apt-get clean`

You should verify that the clean commands actually do something useful before permanently including them in your `Dockerfile`.

To make it easier to avoid copying things from the host into container that you do not need, having a well-crafted [`.dockerignore`](https://docs.docker.com/engine/reference/builder/#dockerignore-file) can help tremendously. Also, having separate folders for the source code and various other files can make things simpler.

### Do not store things in cache if you do not need to
A lot of tools use a cache. This might not be ideal for your situation. If you only need something once, do not cache it. So:
* `pip install --no-cache-dir`
* `apk add --no-cache`

For `apt-get`, you can either configure `apt` to not cache, or you could delete `/var/cache/apt/archives` (the default location for the apt cache).

### For build-time arguments, use build-time arguments and not environment variables
Use `ARG` instead of `ENV`, if possible. The former makes a variable available to `docker` during `build`, the latter permanently becomes an environment variable. Usually, environment variables should be part of the _configuration_, not the build process. You can override default build arguments with `--build-arg variable=value` and environment variables with . Note that the variable you pass in will `not` be shown in the `docker build` output. Feel free to do things like
```
docker build --build-arg HTTP_PROXY=http://10.20.30.2:1234 .
```
if you want to. If you want to overwrite an environment variable during `docker build`, you have to do this
```
ARG variable_name
ENV environment_variable_var_name=$var_name
```

and then
```
docker build --build-arg environment_variable=1234 .
```

* Reduce the verbosity of your tools! Figure out the correct level of verbosity of your tools, and use it! For `apt-get`, for example, it is common to pass `-yqq`, meaning "yes" and "quiet-quiet". It is simply a matter of searching the web for "[tool] reduce output" or something similar.

## Maven
Maven can require a bit of special care. There are also some tricks along the way that can be useful.

### Consider using a Maven image
Building with different Maven versions can easily become a source of confusion. To make sure you get a certain Maven version, consider using a [Maven](https://hub.docker.com/_/maven) image instead of installing Maven into your image. You can "lock" both the Java version and the Maven version in the image name if you use the Maven image instead of the "pure" JDK image. And, you save one install step.

### Require a minimum Maven version
In order to make sure `mvn` commands run outside Docker are run with at least a minimum version, add this to your `<build><plugins>...</plugins></build>`:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-enforcer-plugin</artifactId>
    <version>3.0.0-M2</version>
    <executions>
        <execution>
            <id>enforce-maven</id>
            <goals>
                <goal>enforce</goal>
            </goals>
            <configuration>
                <rules>
                    <requireMavenVersion>
                        <version>3.5.0</version>
                    </requireMavenVersion>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
``` 

In this case, attempting to build the project with a Maven version lower than 3.5.0 will result in an error. This is great for build stability! And, while you're at it, put these in your `<properties>...</properties>`:
```
<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
```

### Reduce verbosity
Use `--batch-mode` (`-B`) to enable "batch mode". This will make Maven output a lot less. It is recommended that you set the following:
```
ARG MAVEN_CLI_OPTS="--batch-mode --quiet"
ARG MAVEN_OPTS="-Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=warn -Dorg.slf4j.simpleLogger.showDateTime=true -Djava.awt.headless=true"
```
This will make Maven fairly quiet, and will not output a progress bar for downloads, and so on.

The variable `${MAVEN_CLI_OPTS}` should always be invoked. For example:
`RUN mvn ${MAVEN_CLI_OPTS} install`
For brevity, we omit `${MAVEN_CLI_OPTS}` in examples that follow.

### Make sure there is a solution for caching in place to avoid builds that take forever
* Either configure a shared cache, have a periodically build (base) image which contains the cache, or accept long download times.
* Also: use the Artifactory (or equivalent); do not download from the regular Maven central.

### In general, try to address all Maven warnings
They are annoying, but usually they indicate something that threaten the stability of your build. The build should not be dependent on any environment, even if it is the defined in the `Dockerfile`!

### Vulnerability scan
Perhaps the easiest way to get going with security scanning, is to use the [`dependency-check`](https://jeremylong.github.io/DependencyCheck/dependency-check-maven/) plugin:
```shell
# Copy the suppression file (remove this line if you do not have one)
COPY dependency-check-suppressions.xml dependency-check-suppressions.xml
RUN mvn dependency-check:check
```

If you want to get a more detailed report in the case of a failed security scan:
```
RUN mvn versions:display-dependency-updates
RUN mvn versions:display-plugin-updates
RUN mvn dependency-check:update-only
COPY dependency-check-suppressions.xml dependency-check-suppressions.xml
RUN mvn dependency-check:check
```

Note that `dependency-check:check` will trigger a update if the cached version is too old (set by the `<cveValidForHours>` in the `<configuration>` section of `dependency-check-maven`.

### Analyze any dependency upgrades
You have two main tools at your disposal. First, to see where each dependency comes from:

```
mvn dependency:tree -Dverbose
```

Look for conflicts in particular. Conflicts here can easily cause weird runtime errors (`ClassNotFoundException`).

You _can_ use `dependency:analyze`, but take its output with a grain of salt. Read more about this plugin on the official [documentation](https://maven.apache.org/plugins/maven-dependency-plugin/).

## Tests as a stage in the `Dockerfile`
```
FROM maven:3.6-jdk-8 as src
COPY src src

FROM src as test
RUN mvn test

FROM src as package
RUN mvn package
```

To run the tests:
```
docker build --pull --target=test .
```

This will not only run the tests, but it will also make sure that the `src` folder is already copied when building the final image. When running
```
docker build --pull .
```
you will see that the first layer is cached. This might not be a big deal in this case, but you get the idea.

## Security scan as phases in the `Dockerfile`
Your `Dockerfile` should include security scanning. The easiest way is to have it as a separate stage, and expand it when needed. It might make sense to have an "outdated dependency" check that is run before the security scan. That way, if the security scan fails, the output from the outdated dependency list will also be included (which might make it more "motivating" to update more than the dependencies that triggered the security scan failure). It is usually impractical to make a pure "outdated dependency check" fail the build.

## Remember to _not_ use any cache when running (scheduled) security scans!
For scheduled dependency check jobs, it is recommended to _not_ use any cache. You definitely do not want an outdated security scan result!
 
```
# Pull the latest version of the base image, and run the dependency check without using any cache. 
docker build --pull --no-cache --target=dependency_check .
```

One could of course clear the _entire_ Docker cache regularly, but this is more self-contained and explicit.

### npm
Use `npm ci` to install from `package-lock.json`. This will (fortunately) fail if it is out of sync with `package.json`.

<pre>
FROM node:latest as frontend-base
RUN npm ci

FROM frontend-base as dependency_check 
RUN pip install safety
RUN safety check --full-report
</pre>

### Python
<pre>
FROM ... as installed_requirements
pip install -r requirements.txt
</pre>

<pre>
FROM installed_requirements as dependency_check
pip install safety
safety check --full-report
</pre>

This will install the latest version of `safety`. Hopefully, you will see something like this:
<pre>
╒══════════════════════════════════════════════════════════════════════════════╕
│                                                                              │
│                               /$$$$$$            /$$                         │
│                              /$$__  $$          | $$                         │
│           /$$$$$$$  /$$$$$$ | $$  \__//$$$$$$  /$$$$$$   /$$   /$$           │
│          /$$_____/ |____  $$| $$$$   /$$__  $$|_  $$_/  | $$  | $$           │
│         |  $$$$$$   /$$$$$$$| $$_/  | $$$$$$$$  | $$    | $$  | $$           │
│          \____  $$ /$$__  $$| $$    | $$_____/  | $$ /$$| $$  | $$           │
│          /$$$$$$$/|  $$$$$$$| $$    |  $$$$$$$  |  $$$$/|  $$$$$$$           │
│         |_______/  \_______/|__/     \_______/   \___/   \____  $$           │
│                                                          /$$  | $$           │
│                                                         |  $$$$$$/           │
│  by pyup.io                                              \______/            │
│                                                                              │
╞══════════════════════════════════════════════════════════════════════════════╡
│ REPORT                                                                       │
│ checked 57 packages, using default DB                                        │
╞══════════════════════════════════════════════════════════════════════════════╡
│ No known security vulnerabilities found.                                     │
╘══════════════════════════════════════════════════════════════════════════════╛
</pre>

You could of course opt to put `safety` in a separate requirements file. For example, one could separate run-time and development-time dependencies into two files.
For example, `requirements-dev.txt` could contain

```
pytest==3.8.0
safety # intentionally omit version to always get the latest version
```

and then `requirements-run.txt` could contain your regular dependencies
```
aioinflux==0.5.1
logstash-formatter==0.5.17
pymongo==3.7.2
...
```

If your team is used to typing `pip install -r requirements.txt`, you can have this as the file contents:
```
-r requirements-run.txt
-r requirements-dev.txt
```

You could then have this in your "build" image
```
WORKDIR /wheels
COPY requirements*.txt .
RUN pip install -r requirements.txt
RUN pip wheel -r requirements-run.txt
```

And, in the resulting production image:
```
COPY --from=requirements /wheels/*.whl /wheels/
RUN pip install /wheels/*.whl && rm -rf /wheels
```

If you want, you can expand the `dependency_check` with
```
pip list --outdated
```

to also get a list of outdated dependencies, or you could of course have it as a separate stage.

# Scanning the Docker images themselves
There can be vulnerabilities coming from various installations

## Clair
For a fairly easy start, use [https://github.com/coreos/clair](Clair). Clair uses a vulnerability database that it keeps updated. It can easily run locally, as part of the build server, and it can run continuously on a server somewhere (keeping the database updated at all times). The template for `config.yaml` only requires a few changes to get started.

### clair-scanner
Though somewhat limited, `clair-scanner` can be used to get started. Point it to an image (that exists locally), and it will output a list of vulnerabilities:
```
./clair-scanner --ip=172.17.0.1 debian:latest

2019/03/01 11:40:59 [INFO] ▶ Start clair-scanner
2019/03/01 11:41:00 [INFO] ▶ Server listening on port 9279
2019/03/01 11:41:00 [INFO] ▶ Analyzing 8ae3ec426609de40c25426f3b621c5aa14a32581c3ddd953874e308495fcf4fa
2019/03/01 11:41:01 [INFO] ▶ Unapproved vulnerabilities [[CVE-2011-4116 CVE-2017-18018 CVE-2016-2781 CVE-2017-7246 CVE-2017-16231 CVE-2017-7245 CVE-2017-11164 CVE-2016-2779 CVE-2017-14062 CVE-2019-7150 CVE-2018-16062 CVE-2018-18520 CVE-2019-7665 CVE-2018-18521 CVE-2019-7148 CVE-2019-7149 CVE-2018-16403 CVE-2018-16402 CVE-2019-7664 CVE-2018-18310 CVE-2018-20482 CVE-2005-2541 CVE-2018-7169 CVE-2007-5686 CVE-2017-12424 CVE-2013-4235 CVE-2018-1000858 CVE-2018-9234 CVE-2018-16869 CVE-2018-10754 CVE-2013-4392 CVE-2018-6954 CVE-2018-15686 CVE-2018-1049 CVE-2017-1000082 CVE-2019-6454 CVE-2018-16888 CVE-2017-18078 CVE-2011-3374 CVE-2009-5155 CVE-2018-6485 CVE-2018-6551 CVE-2017-1000408 CVE-2017-16997 CVE-2017-18269 CVE-2019-9169 CVE-2015-8985 CVE-2010-4756 CVE-2018-1000001 CVE-2016-10739 CVE-2010-4052 CVE-2017-15671 CVE-2017-1000409 CVE-2017-12132 CVE-2018-20796 CVE-2016-10228 CVE-2018-11237 CVE-2017-15670 CVE-2019-7309 CVE-2017-15804 CVE-2010-4051 CVE-2019-6488 CVE-2018-11236 CVE-2018-6829]]
```

#### Configuration
`clair-scanner` accepts a whitelist file.

### Limitations of `clair-scanner` CLI tool
* It scans all layers, and simply adds up all the vulnerabilities.
  * Typically Docker images are written in a way that this makes sense, but not always.
* It does not print the vulnerability descriptions, only the IDs of the vulnerabilities. You will have to search the web for them yourself.

Clair does support both of these things, however, through other programs, as well as through the API directly.

### Klair
You can use [Klair](https://github.com/optiopay/klar) for more flexibility. It supports, among other things:
* A vulnerability threshold
* Outputting the vulnerability descriptions directly

### API
For getting the description of a given vulnerability, and to pass in a layer reference instead of an entire image, you can use the API.

There exists various scripts on GitHub that use the API and creates nice reports. You can try them out, but it is also fairly easy to "roll your own".

## Nuclear option: remove something from image, then copy the root file system in the final step
Traditionally, Docker images are written in a way which only _adds_ things to the image. If something is only needed for one step, it is usually removed the same layer:
```Dockerfile
RUN apt-get update && apt-get install gcc --no-install-recommends && make && apt-get -y purge gcc
 ```
With Docker multi-stage builds, the output from `make` could simply be copied to a "new" image (see `Java/Dockerfile`).

```Dockerfile
FROM large-and-insecure-image as trimmed
RUN build
RUN remove something present in the original image (apt-get purge or equivalent)

FROM scratch
COPY --from=trimmed / /
```

# Simple checklist
* Are you using at least one tool for security analysis that are specific to the technology your application is developed with?
* Will (extra and) scheduled security scans happen independent of development?
* Will a failing security scan block a production deployment?
* Are you using a minimal base image for the resulting image of `docker build .`?
* Is the docker image size reasonable?
* Run `find /` inside the container. Is there something worth removing, not copying/installing to begin with, or adding to `.dockerignore`?
* Does Clair report any relevant vulnerabilities?
* Is the build time  reasonable?
* Are you copying requirements definitions (`pom.xml`, `requirements.txt`) and installing dependencies *before copying the code*? (cache utilization)
* Could the Dockerfile be more readable without sacrificing anything else?
* Does your CI/CD pipeline match closely what your `Dockerfile` has defined? If not, should it?

# Final recommendations
Start with the public-facing services, and at the highest severity. You'll get through most vulnerabilities in no time.
Enforce policies in your build and deploy process.
