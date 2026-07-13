---
title: "Accessing Spring Cloud Bindings' Bindings without Containerizing"
date: 2022-09-13T21:30:12+09:00
categories: ["Spring"]
tags: ["Spring", "Spring Cloud Bindings"]
thumbnail: "2022-08-15T00-46-16.png"
---

This post describes how to use Spring Cloud Bindings without containerizing.<!--more-->
The flow is as follows:

- Set the required System Property
- Set the SERVICE_BINDING_ROOT environment variable
- Update pom.xml
- (Optional) Create your own Bindings

For Service Bindings themselves, see:

https://servicebinding.io/

##  Set the required System Property

When starting Spring Boot, set this System Property:

```
org.springframework.cloud.bindings.boot.enable=true
```

When using IntelliJ or the like, configure it as follows:
`Run → Edit Configurations...` and add the setting below.

![](2022-08-15T00-31-01.png)

## Set the SERVICE_BINDING_ROOT environment variable

Set the starting directory for Service Bindings.

When using IntelliJ or the like, configure it as follows:
`Run → Edit Configurations...` and add the setting below.

![](2022-08-15T00-32-44.png)

## Update pom.xml

Add the required dependencies as follows.
Update the value of `<spring-cloud-bindings.version>` as appropriate.


```xml
  <properties>
    ...
    <spring-cloud-bindings.version>1.10.0</spring-cloud-bindings.version>
    ...
  </properties>
  <dependencies>
      ...
      <dependency>
          <groupId>org.springframework.cloud</groupId>
          <artifactId>spring-cloud-bindings</artifactId>
          <version>${spring-cloud-bindings.version}</version>
      </dependency>
    　...
  </dependencies>
  <repositories>
        ...
        <repository>
            <id>spring-releases</id>
            <url>https://repo.spring.io/release/</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
        ...
  </repositories>
  <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.springframework.cloud</groupId>
                            <artifactId>spring-cloud-bindings</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
  </build>
```
Note that without the block below, containerizing later collides with the Spring Boot buildpack's update behavior and errors out.

```
<configuration>
    <excludes>
        <exclude>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-bindings</artifactId>
        </exclude>
    </excludes>
</configuration>
```

## (Optional) Create your own Bindings

With everything configured so far, you can now create your own Bindings.
For example, prepare code like this:

```java
package jp.vmware.tanzu.twitterwordclouddemo.utils;

import org.jetbrains.annotations.NotNull;
import org.springframework.cloud.bindings.Binding;
import org.springframework.cloud.bindings.Bindings;
import org.springframework.cloud.bindings.boot.BindingsPropertiesProcessor;
import org.springframework.core.env.Environment;

import java.util.List;
import java.util.Map;

public class TwitterBindingsPropertiesProcessor implements BindingsPropertiesProcessor {

	public static final String TYPE = "twitter";

	@Override
	public void process(Environment environment, @NotNull Bindings bindings, @NotNull Map<String, Object> properties) {
		if (!environment.getProperty("jp.vmware.tanzu.bindings.boot.twitter.enable", Boolean.class, true)) {
			return;
		}
		List<Binding> myBindings = bindings.filterBindings(TYPE);
		if (myBindings.size() == 0) {
			return;
		}
		properties.put("twitter.bearer.token", myBindings.get(0).getSecret().get("bearer-token"));
	}

}
```

Additionally, create `META-INF/spring.factories` like this:

```
org.springframework.cloud.bindings.boot.BindingsPropertiesProcessor=jp.vmware.tanzu.twitterwordclouddemo.utils.TwitterBindingsPropertiesProcessor
```

Then place a directory structure like the following into the directory set as SERVICE_BINDING_ROOT, and Service Bindings become active at startup.

![](2022-08-15T00-44-57.png)
