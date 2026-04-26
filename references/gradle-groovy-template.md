# Gradle Groovy Template Guide

This guide defines the canonical Gradle build files for all generated projects.
Always start from the org template in `references/org-gradle-files/` and merge
generated content into it using the rules below.

---

## Merge Rules (CRITICAL)

1. **Never remove** org-defined repositories, plugins, or version constraints
2. **Add** generated dependencies into the appropriate existing dependency block
3. **Do not duplicate** dependencies already present in org template
4. **Preserve** all org plugin applications (checkstyle, jacoco, sonar, etc.)
5. **Preserve** org repository definitions (Nexus, Artifactory, Maven Central order)
6. If the org template defines a `dependencyManagement` block (Spring BOM), do NOT
   add explicit versions to managed dependencies

---

## Dependency Catalogue

Add only the dependencies relevant to what is being generated.

### Always Required
```groovy
// Spring Boot core
implementation 'org.springframework.boot:spring-boot-starter-web'
implementation 'org.springframework.boot:spring-boot-starter-validation'
implementation 'org.springframework.boot:spring-boot-starter-security'
implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
implementation 'org.springframework.boot:spring-boot-starter-actuator'

// Lombok
compileOnly 'org.projectlombok:lombok'
annotationProcessor 'org.projectlombok:lombok'
testCompileOnly 'org.projectlombok:lombok'
testAnnotationProcessor 'org.projectlombok:lombok'
```

### Testing (always required)
```groovy
testImplementation 'org.springframework.boot:spring-boot-starter-test'
testImplementation 'org.springframework.security:spring-security-test'
```

### Database — H2 (dev profile, add when DB layer requested)
```groovy
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
runtimeOnly 'com.h2database:h2'
```

### Database — PostgreSQL (prod, add when DB layer requested)
```groovy
runtimeOnly 'org.postgresql:postgresql'
```

### Org SDK JAR (add when SDK JAR supplied by user)
```groovy
implementation fileTree(dir: 'libs', include: ['*.jar'])
```

---

## Canonical build.gradle (fallback if no org template exists)

Use this ONLY if no org Gradle template is baked into the skill.

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.0'
    id 'io.spring.dependency-management' version '1.1.4'
}

group = 'com.example'             // replaced with user's base package root
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '21'

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

dependencies {
    // Spring Boot
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'

    // Lombok
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    // Testing
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.security:spring-security-test'
    testCompileOnly 'org.projectlombok:lombok'
    testAnnotationProcessor 'org.projectlombok:lombok'
}

tasks.named('test') {
    useJUnitPlatform()
}

// Java 21 compiler options
tasks.withType(JavaCompile).configureEach {
    options.compilerArgs += ['--enable-preview']  // only if preview features used
    options.release = 21
}
```

---

## Canonical settings.gradle

```groovy
rootProject.name = '{app-name}'    // replaced with user's artifact name
```

---

## Canonical gradle-wrapper.properties

```properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.13-bin.zip
networkTimeout=10000
validateDistributionUrl=true
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

---

## Dependency Version Policy

- Spring Boot manages versions for all `org.springframework.*` and common libraries
  via the Spring BOM — do NOT add explicit versions to these
- Lombok version is managed by Spring Boot BOM in Spring Boot 3.x — no explicit version needed
- H2 and PostgreSQL versions are managed by Spring BOM — no explicit version needed
- Only add explicit versions for libraries NOT managed by Spring BOM

---

## Lombok Annotation Processor — Critical Setup

Lombok requires both `compileOnly` AND `annotationProcessor` declarations.
Without the `annotationProcessor` line, Lombok annotations will not be processed
and the build will fail with "cannot find symbol" errors.

For Java 21 specifically, ensure:
```groovy
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}
```

This is preferred over `sourceCompatibility` for Gradle 8.x projects.

---

## Merging Generated Dependencies Into Org Template

When the org template already has a `dependencies {}` block:

1. Identify which of the "Always Required" deps are already present → skip those
2. Append missing deps into the correct configuration block (implementation, compileOnly, etc.)
3. For DB deps — only add if user requested DB layer
4. For SDK JAR dep — only add if user supplied a JAR
5. Never re-order existing deps — append at the end of each configuration group

Example — org template has:
```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    // ... other org deps
}
```

After merge for a project with DB + JWT:
```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    // ... other org deps (untouched)

    // Generated dependencies
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'com.h2database:h2'
    runtimeOnly 'org.postgresql:postgresql'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.security:spring-security-test'
    testCompileOnly 'org.projectlombok:lombok'
    testAnnotationProcessor 'org.projectlombok:lombok'
}
```