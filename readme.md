# Renovate bug reproduction

## What

In my prod repository, I've started seeing "Renovate failed to look up the following dependencies", followed by a list of dependencies that are not in public maven repositories. Some of these are very old jars, some are published privately from other corporate projects.

In my `build.gradle`, I specify our private maven repository, which is a Nexus Repository Manager that proxies Maven Central and also includes both our ancient jars and other corporate-published jars.

```
repositories {
    // Funnel all mavening through a central internal repo, for caching and injecting of old jars not otherwise available via maven
    maven {
        name "My Internal Maven Repo"
        url = 'https://repository.timtest.orf/repository/maven-public'
    }
}
```

## This is fake and Gradle knows it

Running `gradle build` in this directory attempts to use that fake repository *instead of Maven Central*:
```
> Task :war FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':war'.
> Could not resolve all files for configuration ':runtimeClasspath'.
   > Could not resolve org.codehaus.groovy:groovy:2.5.16.
     Required by:
         project :
      > Could not resolve org.codehaus.groovy:groovy:2.5.16.
         > Could not get resource 'https://repository.timtest.orf/repository/maven-public/org/codehaus/groovy/groovy/2.5.16/groovy-2.5.16.pom'.
            > Could not GET 'https://repository.timtest.orf/repository/maven-public/org/codehaus/groovy/groovy/2.5.16/groovy-2.5.16.pom'.
               > repository.timtest.orf: nodename nor servname provided, or not known
```

Here we see that Gradle is using only this repository URL to locate dependencies.

## Stop, Renovate, and Listen

Running Renovate against this project results in an [onboarding PR](https://github.com/timkingman/renovate-gradle-bug-20220930/pull/1) and a list of the dependencies that will be updated after onboarding.

From the Renovate logs:
```
DEBUG: Matched 1 file(s) for manager gradle: build.gradle
DEBUG: Found gradle package files
DEBUG: Found 1 package file(s)
INFO: Dependency extraction complete
{
  "baseBranch": "main",
  "stats": {
    "managers": {
      "gradle": {
        "fileCount": 1,
        "depCount": 6
      }
    },
    "total": {
      "fileCount": 1,
      "depCount": 6
    }
  }
}
DEBUG: Looking up org.codehaus.groovy:groovy in repository https://repo.maven.apache.org/maven2/
DEBUG: Looking up org.codehaus.groovy:groovy-json in repository https://repo.maven.apache.org/maven2/
DEBUG: Looking up org.springframework:spring-web in repository https://repo.maven.apache.org/maven2/
...
DEBUG: http statistics
{
  "urls": {
    "https://api.github.com/graphql (POST,200)": 2,
    "https://api.github.com/repos/timkingman/.github/contents/renovate-config.json (GET,404)": 1,
    "https://api.github.com/repos/timkingman/renovate-config/contents/default.json (GET,404)": 1,
    "https://api.github.com/repos/timkingman/renovate-config/contents/renovate.json (GET,404)": 1,
    "https://api.github.com/repos/timkingman/renovate-gradle-bug-20020930/git/commits (POST,201)": 1,
    "https://api.github.com/repos/timkingman/renovate-gradle-bug-20020930/git/refs (POST,201)": 1,
    "https://api.github.com/repos/timkingman/renovate-gradle-bug-20020930/git/trees (POST,201)": 1,
    "https://api.github.com/repos/timkingman/renovate-gradle-bug-20020930/pulls (GET,200)": 1,
    "https://api.github.com/repos/timkingman/renovate-gradle-bug-20020930/pulls (POST,201)": 1,
    "https://api.github.com/repos/whitesource/merge-confidence/contents/beta.json (GET,200)": 1,
    "https://repo.maven.apache.org/maven2/org/codehaus/groovy/groovy-json/3.0.13/groovy-json-3.0.13.pom (GET,200)": 1,
    "https://repo.maven.apache.org/maven2/org/codehaus/groovy/groovy-test/3.0.13/groovy-test-3.0.13.pom (GET,200)": 1,
    "https://repo.maven.apache.org/maven2/org/codehaus/groovy/groovy-test/maven-metadata.xml (GET,200)": 1,
    "https://repo.maven.apache.org/maven2/org/codehaus/groovy/groovy/3.0.13/groovy-3.0.13.pom (GET,200)": 1,
    "https://repo.maven.apache.org/maven2/org/springframework/spring-test/5.3.23/spring-test-5.3.23.pom (GET,200)": 1,
    "https://repo.maven.apache.org/maven2/org/springframework/spring-web/5.3.23/spring-web-5.3.23.pom (GET,200)": 1,
    "https://repo.maven.apache.org/maven2/org/springframework/spring-webmvc/5.3.23/spring-webmvc-5.3.23.pom (GET,200)": 1
  },
  "hostStats": {
    "api.github.com": {
      "requestCount": 11,
      "requestAvgMs": 301,
      "queueAvgMs": 0
    },
    "repo.maven.apache.org": {
      "requestCount": 7,
      "requestAvgMs": 725,
      "queueAvgMs": 0
    }
  },
  "totalRequests": 18
}
DEBUG: dns cache
{
  "hosts": [
    "api.github.com",
    "repo.maven.apache.org"
  ]
}
```

My internal repository URL appears nowhere in the debug logs, and Renovate appears to only hit repo.maven.apache.org.

## Expected

I expect Renovate to *only* use my internal repository to resolve artifacts, because that's what Gradle does. I'm not entirely sure that Renovate ever actually did this, but this "Renovate failed to look up" error seems fairly recent. (Though it's possible I haven't noticed in months.)

## Guesses

https://github.com/renovatebot/renovate/pull/14783/files suggests that maybe the issue is giving my repository a `name` line above its `url`. I'm sure I got this from some gradle docs, possibly https://docs.gradle.org/current/userguide/declaring_repositories.html#sec:handling_credentials .
