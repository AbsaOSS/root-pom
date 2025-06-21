# Root POM for ABSA OSS repos

[![Maven Central](https://maven-badges.herokuapp.com/maven-central/za.co.absa/root-pom/badge.svg)](https://search.maven.org/search?q=g:za.co.absa%20AND%20a:root-pom)

The root POM defines a set of profiles and properties that implement conventions on how the ABSA OSS Maven based projects are built, released,
published, checked for license or otherwise treated in the CI/CD pipelines.

The profiles are designed to cover independent concerns, so they can be used separately or combined in a single Maven run.

Profiles should be activated using conditional approach by specifying `-D` argument. (Don't use `-P`)

Example:
```shell
mvn ... -Dmy-profile
```

## Setup

Configure your project POM

```xml

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <parent>
        <groupId>za.co.absa</groupId>
        <artifactId>root-pom</artifactId>
        <version>x.y.z</version>
    </parent>

    <scm>
        <url>https://YOUR_PROJECT_URL_HERE</url>
        <connection>${scm.connection}</connection>
        <developerConnection>${scm.developerConnection}</developerConnection>
        <tag>HEAD</tag>
    </scm>

</project>
```

## Usage

### Create a project release

The release is done with the help of standard Maven release plugin configured in the `release` profile.
(We only use `release:prepare` goal, never `release:perform` that is substituted with other dedicated profiles).

```shell
mvn release:prepare -Drelease -Dscm.developerConnection=scm:git:https://github.com/AbsaOSS/...
```

The SCM credentials can be set up either in the command line `-Dusername=... -Dpassword=...`, or via the `<server>` tag in the Maven `settings.xml`

```xml

<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">

    <servers>
        <server>
            <id>absaoss-scm</id>
            <username>...</username>
            <password>...</password>
        </server>
    </servers>

</settings>
```

This will run all tests, create `release/x.y.z` tag, increment the development version and push everything into the given SCM repo

See [Maven release plugin](https://maven.apache.org/maven-release/maven-release-plugin/usage.html) for details.

### Check for license headers

Simply activate `license-check` profile on any Maven phase.

```shell
mvn validate -Dlicense-check
```  

### Measure code coverage

Simply activate `code-coverage` profile on verify Maven phase.

```shell
mvn verify -Dcode-coverage
```  

See [Maven RAT plugin](https://creadur.apache.org/rat/apache-rat-plugin/index.html)

### Deploy to OSSRH (Maven Central)

Configure **Maven Central Portal** credentials and **GPG passphrase** in the Maven `settings.xml`

```xml

<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">

    <servers>
        <server>
            <id>ossrh</id>
            <!-- See https://central.sonatype.org/publish/generate-portal-token/ -->
            <username>...</username>
            <password>...</password>
        </server>
        <server>
            <id>gpg.passphrase</id>
            <passphrase>...</passphrase>
        </server>
    </servers>

</settings>
```

Import a GPG key to sign the artifacts. If you have several keys imported, the default one will be used.

```shell
gpg --import ...
```

Then run Maven `deploy` phase with the `ossrh` profile enabled

```shell
mvn deploy -DskipTests -Dossrh
```

The staging repository will be released and closed automatically.

### Build Docker images

Using the following template, create a `Dockerfile` in a module for which you want to build a Docker image

```dockerfile

FROM base_docker_image_coordinates

# legal stuff
LABEL \
    vendor="ABSA" \
    copyright="2021 ABSA Group Limited" \
    license="Apache License, version 2.0"
    
# These arguments are propagated from the corresponding Maven properties.
# Uncomment what you need.
#
# ARG PROJECT_NAME
# ARG PROJECT_GROUP_ID
# ARG PROJECT_ARTIFACT_ID
# ARG PROJECT_VERSION
# ARG PROJECT_BASEDIR
# ARG PROJECT_BUILD_DIRECTORY
# ARG PROJECT_BUILD_FINAL_NAME    

# The rest of your Dockerfile here

```

In the corresponding `pom.xml`, specify the Docker image name and enable the dockerfile Maven plugin by setting the `<skip>` property to `false`

```xml

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <properties>
        <docker.imageName>my-docker-image</docker.imageName>
    </properties>

    <build>
        <plugins>
            <plugin>
                <groupId>io.fabric8</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <configuration>
                    <skip>false</skip>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

To build the Docker image execute Maven `install` phase with the `docker` profile enabled. You also need to specify the
mandatory `docker.repositoryUrl`. It serves as the image name prefix, and is mandatory even if you don't want to push the image in any remote
repo.

```shell
mvn install -Ddocker -Ddocker.repositoryUrl=foo
```

The above example will create a Docker image with the name `foo/my-docker-image` and two tags - `latest` and `x.y.z` (corresponding the POM version)

##### Push the image to a remote repository
Specify `docker.repositoryUrl` property accordingly and execute Maven `deploy` phase.

The example command below with create an image and push into the AbsaOSS space on the Docker Hub.
```shell
mvn deploy -Ddocker -Ddocker.repositoryUrl=docker.io/absaoss
```

##### Tweaking image names and tags

See the `<docker.*>` properties in the root `pom.xml` for details. All those parameters can be set/overwritten in the command line.

```shell
mvn ... -Dxxx.yyy=zzz
```

## License

---

    Copyright 2021 ABSA Group Limited
    
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at
    
        http://www.apache.org/licenses/LICENSE-2.0
    
    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.
