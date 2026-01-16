Gradle
======

## Tutorial

- <https://docs.gradle.org/current/userguide/part1_gradle_init.html>
  - `gradle init` for basic build structure generates
    - `settings.gradle` contains `rootProject.name`
    - `build.gradle` is empty
    - gradle wrapper
      - it consists of `gradlew`, `gradlew.bat`, and `gradle`
      - a user can use the wrapper without installing gradle first
        - the wrapper automatically downloads a copy of gradle
      - `gradle wrapper --gradle-version latest` updates the wrapper to download
        the latest version
        - it updates `gradle/wrapper/gradle-wrapper.properties` to point to the
          latest release
  - `gradle init` for an java app generates
    - `settings.gradle` additionally contains
      - `foojay-resolver` plugin to download JDK automatically
      - `include('app')` to include the subdir
    - no more `build.gradle`
    - `app/build.gradle` is the build config
    - `app/src` is the generated hello world source code
      - `app/src/main/java/org/example/App.java` has two parts
      - `app/src/main/java` is the path
      - `org/example/App.java` is the java class
  - android studio new project generates
    - `gradle.properties` contains
      - `org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8` tweaks gradle
        daemon
      - `android.useAndroidX=true`
      - `android.nonTransitiveRClass=true`
    - `settings.gradle`
      - `repositories` contains `google()`, `mavenCentral()`, and
        `gradlePluginPortal()`
    - `app/build.gradle`
      - `plugins` contains `libs.plugins.android.application`
      - `dependencies`
      - `android` contains android-specific settings
- <https://docs.gradle.org/current/userguide/part2_gradle_tasks.html>
  - `gradle tasks` lists available tasks
  - to add a custom task
    - edit `app/build.gradle`
    - add `tasks.register(..., "taskName") { ... }`
  - `gradle build` builds the app
    - `-i` to increase log level
    - it downloads jdk and deps to `~/.gradle`
    - it builds in `app/build/` dir
    - dependent tasks
      - `:app:compileJava`
      - `:app:processResources`
      - `:app:classes`
      - `:app:jar`
      - `:app:startScripts`
      - `:app:distTar`
      - `:app:distZip`
      - `:app:assemble`
      - `:app:compileTestJava`
      - `:app:processTestResources`
      - `:app:testClasses`
      - `:app:test`
      - `:app:check`
  - `gradle run` runs the app
- <https://docs.gradle.org/current/userguide/part3_gradle_dep_man.html>
  - `app/build.gradle` is the build config
    - `plugins` includes `application`, to build an CLI app
    - `repositories` uses `mavenCentral()`
      - that is, <https://mvnrepository.com/repos/central>
      - `google()` uses <https://maven.google.com/web/index.html>
    - `dependencies` lists java class deps
    - `java` sets the JDK version
    - `application` specifies the main class of the app
  - `gradle :app:dependencies` lists java class dependencies
