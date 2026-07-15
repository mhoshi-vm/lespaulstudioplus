---
title: "Building a Greenplum Dockerfile and Starting It from Testcontainers"
date: "2026-07-16T01:00:00+09:00"
tags: ["Greenplum", "Testcontainers", "Spring"]
thumbnail: "img.png"
---

For application development based on Greenplum, I often used to substitute something like Postgres.
The problem was that it was inconvenient for trying out queries that only exist in Greenplum.
So I developed a Dockerfile that lets you start Greenplum in a container.
As a bonus, I also wired it up with Spring via Testcontainers.
<!--more--> 

## Introduction

Greenplum is an RDB based on Postgres.
Running an app developed on Postgres against Greenplum is usually fine, but it becomes a problem when you want to develop based on queries that only Greenplum has.

Until now, the main workflow was to build a test environment and verify things there, but I wondered whether it could be done in a container — so I built one.

## Code

It's here. The README also has detailed steps, so refer to that.
It includes extensions such as PostGIS and MADlib, as well as features like PXF.

[https://github.com/mhoshi-vm/greenplum-learn/tree/main/docker/gp78](https://github.com/mhoshi-vm/greenplum-learn/tree/main/docker/gp78)

Note that this Dockerfile is strictly for verification purposes, so please don't use it for things like production.

## Trying It with Spring's Testcontainers

Once you have a Docker image, you can also call it from Spring's Testcontainers.

You need these dependencies in pom.xml.

```
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-testcontainers</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>testcontainers-postgresql</artifactId>
  <scope>test</scope>
</dependency>
```

Then add code like this.

```java
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.context.annotation.Bean;
import org.testcontainers.containers.wait.strategy.Wait;
import org.testcontainers.postgresql.PostgreSQLContainer;
import org.testcontainers.utility.DockerImageName;

@TestConfiguration(proxyBeanMethods = false)
class GreenplumTestcontainersConfiguration {

	@Bean
	@ServiceConnection
	PostgreSQLContainer postgresContainer() {
		return new PostgreSQLContainer(DockerImageName.parse("gp7").asCompatibleSubstituteFor("postgres"))
			.waitingFor(Wait.forLogMessage("^.*statement: GRANT ALL PRIVILEGES ON DATABASE.*\\n", 1))
			.withReuse(true);

	}

}
```

## Conclusion

If you can start Greenplum with Docker, it's convenient to use it with Testcontainers.
