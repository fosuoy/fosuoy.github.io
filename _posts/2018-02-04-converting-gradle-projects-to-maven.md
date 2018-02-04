---
layout: post
title: "Converting Gradle project dependencies to Maven's xml syntax"
date: 2018-02-04
---

This is a (very) short article about converting your projects from Maven to Gradle.

<br>
Maven / Gradle are the most popular dependency managers for Java projects, Gradle
is the more straightforward to use, Maven is the more widely supported.

<br>
Occasionally you might want to convert a project managed by Gradle over to Maven.

<br>
The main issue around it is converting from Gradle's Groovy syntax for dependencies
over to Maven's XML.

<br>
A snippet found on the internet details that adding the following to your build.gradle:


```
task writeNewPom << {
    pom {
        project {
            inceptionYear '<INSERT YEAR HERE>'
            licenses {
                license {
                    name 'The Apache Software License, Version 2.0'
                    url 'http://www.apache.org/licenses/LICENSE-2.0.txt'
                    distribution 'repo'
                }
            }
        }
    }.writeTo("pom.xml")
}
```


Will convert your gradle project over to a `pom.xml`!


Going from there you can define your build / install / compile steps as you like.
