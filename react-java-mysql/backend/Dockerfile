# syntax=docker/dockerfile:1.4

FROM --platform=$BUILDPLATFORM maven:3.8.5-eclipse-temurin-17 AS builder
WORKDIR /workdir/server
COPY pom.xml /workdir/server/pom.xml
#resolves all project dependencies, including plugins and reports and their dependencies
RUN mvn dependency:go-offline
#copy server's sources
COPY src /workdir/server/src
# mvn install frames a dependency tree based on the project configuration pom.xml on all the sub projects
# under the super pom.xml (the root POM, if any) and downloads/compiles all the needed components in a directory
# called .m2 under the user's folder.
RUN mvn install

FROM builder as dev-envs
# update os and install git
RUN <<EOF
apt-get update
apt-get install -y git
EOF
# Create an user (vscode), then create a group docker finally add the user to the group
RUN <<EOF
useradd -s /bin/bash -m vscode
groupadd docker
usermod -aG docker vscode
EOF
# install Docker tools (cli, buildx, compose)
# When you build a Docker image, each instruction in the Dockerfile creates a new layer on top of the previous one
# gloursdocker/docker this is the official docker image which has all the base files required for the application
# so here we are copying the entire contents of the gloursdocker/docker to our docker image
# as there is already an image of gloursdocker/docker which is official image of docker
# we need not build the same image again and used the cached one from gloursdocker/docker.
# The root directory was also copied to include the entire contents of that base image as 
# the foundation for the new Docker image being built.
COPY --from=gloursdocker/docker / /
CMD ["mvn", "spring-boot:run"]
# Preparing production: create directory and pack all into a jar file.
FROM builder as prepare-production
RUN mkdir -p target/dependency
WORKDIR /workdir/server/target/dependency
RUN jar -xf ../*.jar
# Specify the jre image to use, the port and the volume
# Copy the dependence
# Specify into Entrypoint -cp to define the class search path of directories and zip/jar files.
FROM eclipse-temurin:17-jre-focal

EXPOSE 8080
VOLUME /tmp
ARG DEPENDENCY=/workdir/server/target/dependency
COPY --from=prepare-production ${DEPENDENCY}/BOOT-INF/lib /app/lib
COPY --from=prepare-production ${DEPENDENCY}/META-INF /app/META-INF
COPY --from=prepare-production ${DEPENDENCY}/BOOT-INF/classes /app
ENTRYPOINT ["java","-cp","app:app/lib/*","com.company.project.Application"]