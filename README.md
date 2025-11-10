<!--
  Licensed to the Apache Software Foundation (ASF) under one
  or more contributor license agreements.  See the NOTICE file
  distributed with this work for additional information
  regarding copyright ownership.  The ASF licenses this file
  to you under the Apache License, Version 2.0 (the
  "License"); you may not use this file except in compliance
  with the License.  You may obtain a copy of the License at
 
   http://www.apache.org/licenses/LICENSE-2.0
 
  Unless required by applicable law or agreed to in writing,
  software distributed under the License is distributed on an
  "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
  KIND, either express or implied.  See the License for the
  specific language governing permissions and limitations
  under the License.
-->

# Gradle - no wrapper jar demo repo

The ASF Release Policy [mentions](https://www.apache.org/legal/release-policy.html#source-packages)
that _a source release SHOULD not contain compiled code._
Jar files, like the `gradle/wrapper/gradle-wrapper.jar` file, are compiled code and should not be
included in source releases.

The Gradle build tool currently offers no way out-of-the box to run Gradle without having the wrapper
jar in the source tree.

This repository demonstrates how to run Gradle without having the wrapper jar in the source tree
with a few simple steps.

## Steps to remove the `gradle-wrapper.jar` from a source repository

1. Copy the files `gradle/gradlew-include.sh` and `gradle/gradlew-include.bat` from this repo
   to the `gradle/` directory of your project.
2. Adopt `gradlew` shell script (POSIX: Linux, macOS, et al.), if present, by add the following line
   ```
   . ${APP_HOME}/gradle/gradlew-include.sh
   ```
   right after the existing assignment (around line 90 in `gradlew`) of `APP_HOME=`.
   Add some empty lines around the added line for visibility.
3. Adopt `gradlew.bat` batch script (Windows), if present, by add the following line
   ```
   powershell -file "%APP_HOME%\gradle\gradlew-include.ps1"
   ```
   right after the existing assignment (around line 35 in `gradlew.bat`) of `APP_HOME=`.
   Add some empty lines around the added line for visibility.
4. Add the following lines to `.gitignore`
   ```gitignore
   # Ignore Gradle wrapper jar file and checksum files
   **/gradle/wrapper/gradle-wrapper.jar
   **/gradle/wrapper/gradle-wrapper-*.sha256
   ```
5. Add the changes to Git
   ```bash
   git add .gitignore
   ```
6. Remove the `gradle-wrapper.jar` file from the source tree.
   In Git:
   ```bash
   git rm -f gradle/wrapper/gradle-wrapper.jar
   ```
7. Run `./gradlew` or `gradlew.bat` as you are used to. No changes to CI are needed.

## Project using this approach

* Apache Polaris (incubating), since donation of the project
* Apache RAT, in the Gradle plugin PRs, currently in draft state
* Various other non-Apache open source projects

## The first Gradle run

The first Gradle run without a `gradle-wrapper.jar` file in the source tree will look like this:
```
$ ./gradlew tasks
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   167  100   167    0     0   1486      0 --:--:-- --:--:-- --:--:--  1491
100    64  100    64    0     0    276      0 --:--:-- --:--:-- --:--:--   276
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 45633  100 45633    0     0   997k      0 --:--:-- --:--:-- --:--:-- 1012k

> Task :tasks

...
```

The two outputs before the `> Task :tasks` line are the outputs of `curl` downloading the checksum file for the
Gradle wrapper jar and the Gradle wrapper jar itself.

A second run will look as you are used to. As the Gradle wrapper jar is already present and matches the checksum,
no downloads are performed. The output looks like this:

```
$ ./gradlew tasks

> Task :tasks
```

## Background - from `gradlew` to Gradle build

The `gradlew.[bat]` scripts is a thin wrapper to start the `gradle-wrapper.jar` file, which
downloads the Gradle distribution configured in the `gradle/wrapper/gradle-wrapper.properties`
file. It also verifies the integrity of the downloaded Gradle distribution.

The `gradlew-wrapper.jar` then starts the Gradle daemon, which performs the actual build.

## Details about the `gradlew-include.[sh|ps1]` files

The `gradlew-include.[sh|ps1]` files basically ensure that the `gradle-wrapper.jar` file matches the Gradle version
and distribution that is configured in `gradle/wrapper/gradle-wrapper.properties` via the `distributionUrl` property.

For example, `distributionUrl=https\://services.gradle.org/distributions/gradle-9.2.0-all.zip` means Gradle version
9.2.0 including the Gradle sources (those can be useful when developing complex build code or debugging builds).
`distributionUrl=https\://services.gradle.org/distributions/gradle-8.14.3-bin.zip` means Gradle version
8.14.3 without the Gradle sources.

The first step of the `gradlew-include.[sh|ps1]` files is to extract the Gradle version and distribution from the
`distributionUrl` property.
This is for example `9.2.0-all`.
The location of the locally stored checksum file is built with the latter value
for example as `gradle/wrapper/gradle-wrapper-9.2.0-all.jar.sha256`.

If the locally stored checksum file does not exist, it is downloaded from
[services.gradle.org/distributions/](https://services.gradle.org/distributions/).

If the gradle-wrapper-jar is not present, it is downloaded from
[`https://raw.githubusercontent.com/gradle/gradle/...`.

The checksum of the downloaded Gradle wrapper jar is always verified against the locally stored checksum file
before the gradle-wrapper is started.

If the checksums of the locally present or downloaded jar do not match with the one in the checksum file,
the gradle-wrapper will not be started at all.

The verification (or integrity check if you want) of the downloaded Gradle wrapper jar
complies with the
[Gradle documentation](https://docs.gradle.org/current/userguide/gradle_wrapper.html#wrapper_checksum_verification).

As the script downloads the Gradle wrapper jar, and the checksum file from two different sources,
we can be pretty sure that the downloaded Gradle wrapper jar is authentic.

## Upgrading Gradle

When upgrading Gradle, the `distributionUrl` and `distributionSha256Sum` properties in
`gradle/wrapper/gradle-wrapper.properties` need to be updated.
Some Gradle upgrades also come with changes to the `gradlew[.bat]` files.

When upgrading Gradle, make sure that the "include lines" iu the `gradlew[.bat]` files remain intact.
Re-add those lines if necessary.

Renovate and Dependabot have functionalities to update Gradle. These tools may report issues with the
Gradle update.
The cause is that the `gradle-wrapper.jar` is not present in the source tree.
This doesn't prevent, for example, Renovate from creating a Gradle version bump PR, but you might have to
manually amend the PR.
Also take care that the `distributionSha256Sum` property in `gradle/wrapper/gradle-wrapper.properties`
is updated and not removed. If in doubt, check the
[Gradle's checksum reference page](https://gradle.org/release-checksums/).

## FAQ

### Do I need both `gradlew` and `gradlew.bat` and the corresponding include files?

Not necessarily.
If you only build on Linux and/or macOS, you don't need the `gradlew.bat` file and the
corresponding include file `gradlew-include.ps1`.
And vice versa.

### Why are there two checksums?

The one checksum is to verify the integrity of the downloaded Gradle wrapper jar.
The other checksum specified via the `distributionSha256Sum` property in `gradle-wrapper.properties`is to verify the
integrity of the Gradle distribution, which is downloaded and verified by the Gradle wrapper jar.

### Does this change cause any work for contributors?

Short answer: No.

### Does this change require any changes to CI?

Short answer: No.

### Advanced topic: Can I use a custom Gradle distribution and/or a custom wrapper jar with this approach?

Changes to the `gradlew-include.[sh|ps1]` files would be needed to make those aware of your customized
Gradle distribution and/or wrapper jar, mostly to adopt the download URLs.

## Additional safety tips

* Set the property `validateDistributionUrl=true` in your `gradle/wrapper/gradle-wrapper.properties` file.
* Configure the property `distributionSha256Sum` to the checksum for your Gradle version and distribution.
  You can get the checksums from [Gradle's checksum reference page](https://gradle.org/release-checksums/), or
  you can download the checksum files from [services.gradle.org/distributions/](https://services.gradle.org/distributions/).
