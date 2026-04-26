---
name: spec-to-springboot
description: >
  Generates a complete, production-ready Java 21 + Spring Boot 3 + Gradle (Groovy) application
  from OAS 3.x (OpenAPI) or RAML 1.0 API specification files. Use this skill whenever the user
  uploads API spec files (YAML/JSON) and wants to generate a Spring Boot application, mentions
  "generate code from spec", "scaffold a Spring Boot project", "create a consumer service",
  "implement an API from OpenAPI/RAML", or wants to produce a runnable IntelliJ IDEA project
  from API contracts. Also triggers for iterative refinement requests like "add pagination to
  the generated endpoint", "update the gateway for the new spec", or "add a new producer".
  Always use this skill when spec files + code generation are mentioned together.
---

# spec-to-springboot Skill

Generates a complete Java 21 + Spring Boot 3 + Gradle 8.13 (Groovy DSL) project from
OAS 3.x / RAML 1.0 specification files. Produces a downloadable ZIP ready to open in IntelliJ IDEA.

---

## Reference Files — Read These As Needed

| File | When to read |
|---|---|
| `references/oas-parsing-guide.md` | When processing OAS 3.x (OpenAPI) spec files |
| `references/raml-parsing-guide.md` | When processing RAML 1.0 spec files |
| `references/code-generation-rules.md` | Before generating ANY Java/Spring code |
| `references/gradle-groovy-template.md` | Before generating build.gradle / settings.gradle |
| `references/org-gradle-files/` | Baked-in org Gradle assets — merge, never replace |

Always read `references/code-generation-rules.md` and `references/gradle-groovy-template.md`
before writing any code. Read the appropriate parsing guide(s) based on detected spec format.

---

## Execution Workflow

### STEP 1 — Detect & Load Specs

1. List all files in `/mnt/user-data/uploads/`
2. For each YAML/JSON file, detect format:
   - First line contains `#%RAML 1.0` → **RAML 1.0** → read `references/raml-parsing-guide.md`
   - File contains `openapi: "3.` or `openapi: 3.` → **OAS 3.x** → read `references/oas-parsing-guide.md`
   - Unknown format → tell the user and stop
3. If multiple spec files uploaded, ask:
   > "Which file is the **consumer** spec (the API your application exposes)?
   > Which are the **producer** specs (APIs your application will call)?"
   - If filenames are self-evident (e.g. `consumer-api.yaml`, `producer-payments.yaml`), infer and confirm.
4. Parse each spec using the appropriate parsing guide. Extract the **Semantic Model** (see below).

#### Semantic Model (extract for every spec)
```
- spec_type: consumer | producer
- format: OAS3 | RAML1
- title: string (from info.title)
- version: string (from info.version)
- base_path: string
- endpoints:
    - path: string
      method: GET|POST|PUT|PATCH|DELETE
      operation_id: string
      path_params: [{name, type, required, constraints}]
      query_params: [{name, type, required, constraints}]
      request_body: {schema_ref, required}
      responses: [{status_code, schema_ref, description}]
      security: [scheme_names]
- schemas: [{name, type, fields: [{name, type, constraints, nullable}]}]
- security_schemes: [{name, type, flows}]
- error_shapes: [{status_code, schema_ref}]
```

---

### STEP 2 — Pre-Generation Clarification Checkpoint

Before generating any code, ask the user the following. Do NOT skip any question.

```
Before I generate your project, I need a few details:

1. **Base Java package** (e.g. `com.acme.orders`)
2. **Application / artifact name** (e.g. `order-service`) — used in settings.gradle and JAR name
3. **Database layer needed?**
   - No — pure API-to-API (no DB)
   - Yes — Spring Data JPA + H2 (dev) / PostgreSQL (prod)
4. **Org SDK JAR supplied?**
   - No — use Spring Boot RestClient (default)
   - Yes — please upload the JAR file and provide: groupId:artifactId:version
5. **Any custom application.yml values?** (e.g. custom port, environment-specific URLs, feature flags)
   - If none, just say "no" and I'll use sensible defaults.
```

Wait for all answers before proceeding.

---

### STEP 3 — Read Reference Files

Based on answers, read:
- `references/code-generation-rules.md` — **always**
- `references/gradle-groovy-template.md` — **always**
- `references/oas-parsing-guide.md` — if any OAS 3.x spec present
- `references/raml-parsing-guide.md` — if any RAML 1.0 spec present
- If SDK JAR supplied AND `org-sdk-guide` skill is available → read `org-sdk-guide/SKILL.md`
  and its relevant reference files before generating Gateway implementations

---

### STEP 4 — Generate Project Files

Generate files in this exact order. Use the semantic models extracted in Step 1.

#### 4a. Gradle Files
Start from the org template in `references/org-gradle-files/`. Merge in all
generated dependencies. Never remove org-defined repositories, plugins, or version constraints.
See `references/gradle-groovy-template.md` for merge rules.

Files to generate:
- `build.gradle` (merged from org template)
- `settings.gradle`
- `gradle/wrapper/gradle-wrapper.properties` (Gradle 8.13)
- `gradlew` and `gradlew.bat` (standard wrapper scripts)
- `libs/{sdk}.jar` placeholder path comment if SDK supplied

#### 4b. Application Entry Point
- `src/main/java/{base-package}/{AppName}Application.java`
  - `@SpringBootApplication`
  - Standard `main` method

#### 4c. Config Classes
- `SecurityConfig.java` — always; JWT resource server; see code-generation-rules.md
- `RestClientConfig.java` — one `@Bean RestClient` per producer spec, baseUrl from `application.yml`
  - If SDK JAR supplied: replace with SDK client bean config per `org-sdk-guide`

#### 4d. Models (from ALL specs)
Follow Java type mapping rules in `references/code-generation-rules.md`.

- `model/request/{Schema}.java` — inbound DTOs from consumer spec request bodies
- `model/response/{Schema}.java` — outbound DTOs from consumer spec responses
- `model/client/{Schema}.java` — DTOs for producer API request/response bodies
- `model/entity/{Entity}.java` — JPA entities (only if DB layer requested)

#### 4e. Controllers (from consumer spec only)
- One controller class per resource group (tag or path prefix)
- `{Resource}Controller.java` in `controller/`
- See code-generation-rules.md for exact annotations and patterns

#### 4f. Services (from consumer spec)
- One interface + impl pair per controller resource
- `service/{Resource}Service.java` — interface
- `service/impl/{Resource}ServiceImpl.java` — stub implementation
- Wires controller → gateway calls

#### 4g. Gateways (one per producer spec)
- `gateway/{ProducerName}Gateway.java` — interface, methods matching producer endpoints called
- `gateway/impl/{ProducerName}GatewayImpl.java` — implementation
  - Default: uses injected `RestClient` bean
  - If SDK supplied: uses SDK client classes per `org-sdk-guide` patterns

#### 4h. Repositories (only if DB layer requested)
- `repository/{Entity}Repository.java` — interface extending `JpaRepository<{Entity}, {IdType}>`
- `repository/impl/` — only if custom queries needed (infer from spec semantics)

#### 4i. Exception Handling
- `exception/GlobalExceptionHandler.java` — `@RestControllerAdvice`
  - Handlers for: `MethodArgumentNotValidException`, `HttpClientErrorException`,
    `HttpServerErrorException`, custom domain exceptions
- `exception/{Domain}Exception.java` — one per major domain error inferred from spec error shapes

#### 4j. application.yml Files
Generate three profiles. See `references/code-generation-rules.md` for structure.
- `application.yml` — base config
- `application-dev.yml` — dev overrides
- `application-prod.yml` — prod overrides

#### 4k. Tests
- `controller/{Resource}ControllerTest.java` — `@WebMvcTest` + `@MockBean` service
- `service/{Resource}ServiceImplTest.java` — `@ExtendWith(MockitoExtension.class)`
- `gateway/{ProducerName}GatewayImplTest.java` — `@RestClientTest`
- Happy path + at least one error/edge case per endpoint

#### 4l. README.md
Always generate. See code-generation-rules.md for required sections.

---

### STEP 5 — Assemble ZIP

```python
import zipfile, os

zip_path = f"/mnt/user-data/outputs/{app_name}.zip"
with zipfile.ZipFile(zip_path, 'w', zipfile.ZIP_DEFLATED) as zf:
    for root, dirs, files in os.walk(f"/home/claude/{app_name}"):
        for file in files:
            abs_path = os.path.join(root, file)
            arc_path = os.path.relpath(abs_path, f"/home/claude/{app_name}")
            zf.write(abs_path, arc_path)
```

Then call `present_files` with the ZIP path.

---

### STEP 6 — Iterative Refinement

When the user requests a change to a previously generated project:

1. **Identify scope** — which files are affected?
   - New/changed endpoint → controller + service interface/impl + tests
   - New/changed producer spec → gateway interface/impl + client models + tests + RestClientConfig
   - DB layer added/changed → repository + entity models + application.yml profiles
   - New producer added → new gateway pair + client models + RestClientConfig bean + tests
2. **Regenerate only affected files** — do not touch unaffected files
3. **Reassemble full ZIP** from previously generated files + newly regenerated files
4. **Deliver fresh ZIP** via `present_files`

If the user uploads a new/updated spec file, re-run Step 1 for that file only, then continue from Step 4 for affected components.

---

## Companion Skill: org-sdk-guide

If the user supplies an org SDK JAR:
1. Check if `org-sdk-guide` skill is available in `available_skills`
2. If available → read `org-sdk-guide/SKILL.md` then its reference files as directed
3. Use SDK patterns for:
   - `RestClientConfig.java` → replaced by SDK client bean config
   - `{ProducerName}GatewayImpl.java` → uses SDK client classes, not raw RestClient
   - `build.gradle` → add `implementation fileTree(dir: 'libs', include: ['*.jar'])`
   - `libs/` folder → include the uploaded JAR in the ZIP
4. If `org-sdk-guide` not available → warn the user:
   > "The org-sdk-guide companion skill is not installed. I'll generate the gateway using
   > Spring Boot RestClient. For SDK-specific patterns, please install org-sdk-guide."

---

## Output Checklist

Before delivering ZIP, verify:
- [ ] `build.gradle` merges org template + generated deps
- [ ] All spec endpoints have a corresponding controller method
- [ ] All producer spec endpoints called have a gateway method
- [ ] All schemas have a corresponding model class
- [ ] `SecurityConfig.java` present and correct
- [ ] `application.yml` + dev + prod profiles all present
- [ ] Tests exist for all controllers and gateways
- [ ] `README.md` present with all required sections
- [ ] ZIP contains the project folder as root (not flat files)
- [ ] If SDK JAR supplied: JAR present in `libs/`, build.gradle references it