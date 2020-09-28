# Jfrog Activation Guardian


A simple Repo, with a croned Circle CI pipeline to maven deploy a single Jar to https://gravitee.jfrog.io/ every week, so the Jfrog server can stay unused without ever get flushed by JFrog, because of no usage


## `Cron` Config

I want the Pipeline to run once a week, On sunday at midnight :

* The Cron configuration for that frequency is :

```ini
0 0 * * 0
```

* test config every 5 minutes:

```ini
*/5 * * * *
```

## Secrets Initialization

```bash
export NAME_OF_ORG="gravitee-lab"
export NAME_OF_REPO_IN_ORG="gravitee-lab/cicd-infra"
secrethub org init "${NAME_OF_ORG}"
secrethub repo init "${NAME_OF_REPO_IN_ORG}"


# --- #
# for the DEV CI CD WorkFlow of
# the Gravitee CI CD Orchestrator
secrethub mkdir --parents "${NAME_OF_REPO_IN_ORG}/dev/jfrog"

# --- #
# for the STAGING CI CD WorkFlow of
# the Gravitee CI CD Orchestrator
secrethub mkdir --parents "${NAME_OF_REPO_IN_ORG}/staging/jfrog"


# --- #
# for the PRODUCTION CI CD WorkFlow of
# the Gravitee CI CD Orchestrator
secrethub mkdir --parents "${NAME_OF_REPO_IN_ORG}/prod/jfrog"

# --- #
# write quay secrets for the DEV CI CD WorkFlow of
# the Gravitee CI CD Orchestrator
export JFROG_CICD_BOT_USERNAME="cicd_bot"
export JFROG_CICD_BOT_SECRET="inyourdreams;)"

echo "${JFROG_CICD_BOT_USERNAME}" | secrethub write "${NAME_OF_REPO_IN_ORG}/dev/jfrog/username"
echo "${JFROG_CICD_BOT_SECRET}" | secrethub write "${NAME_OF_REPO_IN_ORG}/dev/jfrog/password"

echo "${JFROG_CICD_BOT_USERNAME}" | secrethub write "${NAME_OF_REPO_IN_ORG}/staging/jfrog/username"
echo "${JFROG_CICD_BOT_SECRET}" | secrethub write "${NAME_OF_REPO_IN_ORG}/staging/jfrog/password"

echo "${JFROG_CICD_BOT_USERNAME}" | secrethub write "${NAME_OF_REPO_IN_ORG}/prod/jfrog/username"
echo "${JFROG_CICD_BOT_SECRET}" | secrethub write "${NAME_OF_REPO_IN_ORG}/prod/jfrog/password"


export JFROG_USERNAME_DEV=$(secrethub read gravitee-lab/cicd-infra/dev/jfrog/username)
export JFROG_SECRET_DEV=$(secrethub read gravitee-lab/cicd-infra/dev/jfrog/password)

export JFROG_USERNAME_STAGING=$(secrethub read gravitee-lab/cicd-infra/staging/jfrog/username)
export JFROG_SECRET_STAGING=$(secrethub read gravitee-lab/cicd-infra/staging/jfrog/password)

export JFROG_USERNAME_PROD=$(secrethub read gravitee-lab/cicd-infra/prod/jfrog/username)
export JFROG_SECRET_PROD=$(secrethub read gravitee-lab/cicd-infra/prod/jfrog/password)

echo "JFROG_USERNAME_DEV=[${JFROG_USERNAME_DEV}]"
echo "JFROG_SECRET_DEV=[${JFROG_SECRET_DEV}]"

echo "JFROG_USERNAME_STAGING=[${JFROG_USERNAME_STAGING}]"
echo "JFROG_SECRET_STAGING=[${JFROG_SECRET_STAGING}]"

echo "JFROG_USERNAME_STAGING=[${JFROG_USERNAME_STAGING}]"
echo "JFROG_SECRET_STAGING=[${JFROG_SECRET_STAGING}]"


```
* creating secrethub service account with permissions to access secrets in `gravitee-lab/cicd-infra` repo :

```bash
export NAME_OF_REPO_IN_ORG="gravitee-lab/cicd-infra"
secrethub service init "${NAME_OF_REPO_IN_ORG}" --description "Circle CI Service for Gravitee CI CD Orchestrator" --permission read | tee ./.the-created.service.token
```
* Then created a Circle CI Org context `cicd-infra`, and in that context, the `SECRETHUB_CREDENTIAL` env. var. with value, the token in output of the `service init` command

## Packaging and deployment to JFrog


Here is a sample script of what the Circle CI Pipeline basically executes :

```bash

# commandes maven

export GUARDIAN_GIT_URI="https://github.com/gravitee-lab/gravitee-gateway"
mkdir -p "$PWD/guard-shift"
git clone "${GUARDIAN_GIT_URI}" $PWD/guard-shift
cd ${MVN_LAB}

# mapping [ -v "$PWD/target:/usr/src/mymaven/target" ] requires to create a docker image to manage UID GID of linux user inside and outside container
# docker run -it --rm -v "$PWD":/usr/src/mymaven -v "$HOME/.m2":/root/.m2 -v "$PWD/target:/usr/src/mymaven/target" -w /usr/src/mymaven maven mvn clean package

# To run mvn clean package:
# [-v "$PWD":/usr/src/mymaven] :  maps the source code inside container
# [-v "$HOME/.m2":/root/.m2] : maps the maven [.m2] on my workstation to the one inside the container. I will ust this one to use settings.xml
# docker run -it --rm -v "$PWD":/usr/src/mymaven -v "$HOME/.m2":/root/.m2 -w /usr/src/mymaven maven mvn clean package
# To run mvn clean package release :
# all gravtiee java pom projects are [https://github.com/gravitee-io/gravitee-parent/]

export JFROG_BUILD_NUMBER='158468578'
#
# see https://github.com/jfrog/project-examples/tree/master/artifactory-maven-plugin-example
export JFROG_USERNAME_DEV=$(secrethub read gravitee-lab/cicd-infra/dev/jfrog/username)
export JFROG_SECRET_DEV=$(secrethub read gravitee-lab/cicd-infra/dev/jfrog/password)
echo "JFROG_USERNAME_DEV=[${JFROG_USERNAME_DEV}]"
echo "JFROG_BUILD_NUMBER=[${JFROG_BUILD_NUMBER}]"

# echo "JFROG_SECRET_DEV=[${JFROG_SECRET_DEV}]"

export DESIRED_MAVEN_VERSION=3.6.3
export MVN_DOCKER_IMAGE="maven:${DESIRED_MAVEN_VERSION}-openjdk-16 "

echo "Run Maven Clean install to package gravitee gateway from existing maven central repo (Nexus Sonatype, Maven Central)"
export MAVEN_COMMAND="mvn clean install"
echo "MAVEN_COMMAND=[${MAVEN_COMMAND}]"

docker run -it --rm -v "$PWD/guard-shift":/usr/src/mymaven -v "$HOME/.m2":/root/.m2 -w /usr/src/mymaven ${MVN_DOCKER_IMAGE} ${MAVEN_COMMAND}

echo "Run Maven Deploy using JFrog Maven Plugin"
export MAVEN_COMMAND="mvn deploy -Dusername=${JFROG_USERNAME} -Dpassword=${JFROG_SECRET} -Dbuildnumber=${JFROG_BUILD_NUMBER}"
echo "MAVEN_COMMAND=[${MAVEN_COMMAND}]"
docker run -it --rm -v "$PWD/guard-shift":/usr/src/mymaven -v "$HOME/.m2":/root/.m2 -w /usr/src/mymaven ${MVN_DOCKER_IMAGE} ${MAVEN_COMMAND}

```


# JFrog deploy plugin


The example described in this section configures the Artifactory publisher to deploy build artifacts either to the `releases` or the `snapshots` repository of the  https://gravitee.jfrog.io <!-- public OSS --> instance of Artifactory when `mvn deploy` is executed.

```Xml
<build>
    <plugins>
        ...
        <plugin>
            <groupId>org.jfrog.buildinfo</groupId>
            <artifactId>artifactory-maven-plugin</artifactId>
            <version>2.7.0</version>
            <inherited>false</inherited>
            <executions>
                <execution>
                    <id>build-info</id>
                    <goals>
                        <goal>publish</goal>
                    </goals>
                    <configuration>
                        <deployProperties>
                            <gradle>awesome</gradle>
                            <review.team>devops</review.team>
                        </deployProperties>
                        <publisher>
                            <!--<contextUrl>https://oss.jfrog.org</contextUrl>-->
                            <contextUrl>https://gravitee.jfrog.io</contextUrl>
                            <username>someUsername</username>
                            <password>somePassword</password>
                            <repoKey>libs-release-local</repoKey>
                            <snapshotRepoKey>libs-snapshot-local</snapshotRepoKey>
                        </publisher>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

When maven deploy successfully uploaded the maven artifacts to artifactory, we can see them in the artifactory Web UI :

![success mvn deploy artifacts in Artifactory Web UI](./doc/images/MVN_DEPLOY_SUCCESS_ARTIFACTORY_2020-09-27T19-10-22.862Z.png)
