---
name: add-bitdive-spring
description: >
  Step-by-step guide for adding the BitDive profiling and replay libraries to any
  Java or Kotlin Spring Boot project (Maven or Gradle, Spring Boot 2.x or 3.x).
  Also covers creating the required config file and optional replay test class.
---

# Add BitDive to a Spring Boot Project

Use this skill whenever you need to instrument a new service or verify that an existing
service is correctly integrated with BitDive.

---

## Step 1: Detect Project Type

Inspect the project and determine:

| Question | How to detect |
|---|---|
| **Build tool** | `pom.xml` → Maven · `build.gradle` / `build.gradle.kts` → Gradle |
| **Spring Boot version** | Check `<parent>` in `pom.xml` or `plugins` in `build.gradle` |
| **Language** | `src/main/java` → Java · `src/main/kotlin` → Kotlin |

---

## Step 2: Fetch the Latest BitDive Version

> [!IMPORTANT]
> Do not use a hardcoded version. Always fetch the current latest from Maven Central.

```
GET https://central.sonatype.com/api/internal/browse/component/versions?sortField=normalizedVersion&sortDirection=desc&page=0&size=5&filter=namespace:io.bitdive,name:bitdive-producer-spring-3
```

Take the `version` field from the first result.

Artifact to use:

| Spring Boot | Producer artifact | Replay artifact |
|---|---|---|
| 3.x | `bitdive-producer-spring-3` | `bitdive-replay-spring3` |
| 2.x | `bitdive-producer-spring-2` | `bitdive-replay-spring2` |

---

## Step 3: Add the Producer Dependency

### Maven (`pom.xml`)

```xml
<dependency>
    <groupId>io.bitdive</groupId>
    <artifactId>bitdive-producer-spring-3</artifactId>
    <version>LATEST_VERSION</version>
</dependency>
```

### Gradle (Groovy)

```groovy
implementation 'io.bitdive:bitdive-producer-spring-3:LATEST_VERSION'
```

### Gradle (Kotlin DSL)

```kotlin
implementation("io.bitdive:bitdive-producer-spring-3:LATEST_VERSION")
```

---

## Step 4: Create `config-profiling-api.yml`

> [!IMPORTANT]
> Always create a **separate** `src/main/resources/config-profiling-api.yml`.
> **Do NOT** add BitDive settings to `application.yml` or `application.properties`.

### 4a. Find the application package

Locate the class annotated with `@SpringBootApplication` and note its package.
Use the **narrowest** service-specific package, for example:

- ✅ `com.company.orders`
- ✅ `com.company.report`
- ❌ `com.company` (too broad)

### 4b. Required config values

| Field | Meaning |
|---|---|
| `moduleName` | BitDive project/module ID |
| `serviceName` | Application name as shown in BitDive |
| `serverUrl` | BitDive backend URL |
| `token` | Authentication token from the BitDive UI |
| `packedScanner` | Application package(s) for instrumentation |

> [!IMPORTANT]
> `moduleName`, `token`, and `serverUrl` must be provided by the human or taken
> from the existing backend-issued BitDive config for that environment.
> Do **not** invent them, rename them, or change them by default.
>
> Derive only the service-local values from the real project:
> - `serviceName`
> - `packedScanner`

> [!NOTE]
> Use the **real Spring application name** of the service you are configuring.
> Never copy `serviceName` from another service's config.

### 4c. Config file template

```yaml
bitdive:
  monitoring:
    moduleName: MODULE_NAME
    serviceName: SERVICE_NAME
    packedScanner:
      - APPLICATION_PACKAGE
    serverUrl: https://cloud.bitdive.io
    token: TOKEN
```

Keep `serverUrl` exactly as provided for the target environment unless the human
explicitly says that this service should point to a different BitDive backend.
If a local BitDive instance is intentionally in use, see
`bitdive-docker-networking`.

---

## Step 5: (Optional) Add the Replay Dependency

The replay library enables trace-based regression tests.

### Maven

```xml
<dependency>
    <groupId>io.bitdive</groupId>
    <artifactId>bitdive-replay-spring3</artifactId>
    <version>LATEST_VERSION</version>
    <scope>test</scope>
</dependency>
```

---

## Step 6: (Optional) Create Replay Test Class

Create `src/test/java/<main_package>/TestControllerTestAbstract.java`.

> [!NOTE]
> After triggering an endpoint, wait **~30 seconds** for the trace to be indexed
> before querying BitDive or running replay tests.

### Option A — Specific test group UUIDs

Use when you want Maven to run only selected groups.

```java
package MAIN_PACKAGE;

import io.bitdive.replay.ReplayTestBase;
import io.bitdive.replay.dto.ReplayTestConfiguration;
import io.bitdive.replay.dto.ReplayTestUtils;

import java.util.Arrays;
import java.util.List;

class TestControllerTestAbstract extends ReplayTestBase {

    @Override
    protected List<ReplayTestConfiguration> getTestConfigurations() {
        return ReplayTestUtils.fromRestApiWithJsonContentConfigFile(
                Arrays.asList(
                        "UUID-1",  // Description
                        "UUID-2"   // Description
                ));
    }
}
```

### Option B — All service test groups

Use when you want Maven to run every group defined for this service in BitDive.

```java
package MAIN_PACKAGE;

import io.bitdive.replay.ReplayTestBase;
import io.bitdive.replay.dto.ReplayTestConfiguration;
import io.bitdive.replay.dto.ReplayTestUtils;

import java.util.List;

class TestControllerTestAbstract extends ReplayTestBase {

    @Override
    protected List<ReplayTestConfiguration> getTestConfigurations() {
        return ReplayTestUtils.fromRestApiWithJsonContentConfigFileAllTest();
    }
}
```

Replace `MAIN_PACKAGE` with the Spring Boot application package.

---

## Summary Checklist

- [ ] Detected build tool, Spring Boot version, and language
- [ ] Fetched latest BitDive producer version from Maven Central
- [ ] Added producer dependency
- [ ] Located the narrowest application package
- [ ] Created `config-profiling-api.yml` with correct `moduleName`, `serviceName`, `serverUrl`, `token`
- [ ] Kept backend-issued values (`moduleName`, `serverUrl`, `token`) unchanged unless the human explicitly requested a different backend
- [ ] (Optional) Added replay dependency
- [ ] (Optional) Created `TestControllerTestAbstract.java` with correct UUIDs
