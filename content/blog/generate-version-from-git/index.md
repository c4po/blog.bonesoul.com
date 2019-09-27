---
title: Semantic versioning for docker tag from Git tag
date: "2017-05-20T15:01:54Z"
description: Semantic versioning for docker tag from Git tag
---

Define the version or build number is a very important step in a continuous integration and deployment pipeline.

When we build the docker image, we should add the version number as the *tag* to the image. And when we need deploy the docker image, we can refer the version number instead using *latest* tag.

[Semantic versioning guideline](http://semver.org/) is a widely used schema to define versions. In practice, how we generate the version number and keep the version number is still a challenge when we design a CI/CD pipeline to build docker images. In this post, I will introduce a method to generate version number from git tags and integrated with Jenkins and docker build.

The idea is to tag the git repo with a naming convention each time when we start a new build on release branch. The build job will check what is the version number of last build and increase the *z* value of *x.y.z* by 1. The build job will not change *x* or *y* value of the version number. There are 3 major steps in this process,

1. Check the current branch

  We only generate the version number when we build on the *release* branch. So the first step is check what is current build branch.

  ```
String getGitBranch() {
  return script.sh(returnStdout: true, script: 'git name-rev --name-only HEAD').trim().minus("remotes/origin/")
}
```

2. Check if we already have a release tag on current commit

  ```
git describe --tags --exact-match --match release/*
```

3. Find the latest tag

  If we don't have a release tag on current commit, we will search for the latest release tag and generate the new version number based on it.

  ```
git describe --tags --abbrev=0 --match release/* | grep -o "[^\/]*$"
```


Let's put everything together


```
String getApplicationVersion() {
  def tagPrefix = "release"
  def releaseBranch = "master"
  def currentVersion = ""
  def currentBranch =getGitBranch()
  if (currentBranch == releaseBranch) {
    try {
      def currentRelease = script.sh(returnStdout: true, script: "git describe --tags --exact-match --match ${tagPrefix}/*")
      currentVersion = currentRelease.toString().replaceFirst("${tagPrefix}/", "")
    } catch (error) { //  no release tag for commit
      try {
        def command = "git describe --tags --abbrev=0 --match ${tagPrefix}/* | grep -o \"[^\\/]*\$\" "
        def latestTag = script.sh(returnStdout: true, script: command)
        def versionSplit = latestTag.toString().split("\\.")
        currentVersion = "${versionSplit[0]}.${versionSplit[1]}.${versionSplit[2].toInteger() + 1}"
      } catch (error) { // no release tag for this repo
        currentVersion = "0.0.1"
      }
    }
    try {
        script.sh(returnStdout: true, script: "git tag ${tagPrefix}/${currentVersion}")
        script.sh(returnStdout: true, script: "git push --tag")
    } catch (error) { // cannot push tag
      //send error message
    }
  }
  return currentVersion
}
```
