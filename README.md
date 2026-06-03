# Spring Framework 5.3.42 Security Fork

This repository is a self-maintained Spring Framework fork published for projects
that still need a Spring Framework 5.3.x based build with selected security
backports.

It is not an official Spring release. Official Spring Framework artifacts are
published by the Spring team under the `org.springframework` Maven group.

## Repository

- GitHub repository: <https://github.com/duanluan/spring-framework>
- Security branch: <https://github.com/duanluan/spring-framework/tree/5.3.42-security>
- Base source: <https://github.com/spring-projects/spring-framework/tree/v5.3.39>
- Base version: `spring-projects/spring-framework:v5.3.39`
- Published version: `5.3.42`
- Maven group: `io.github.duanluan.springframework`
- Java package names: `org.springframework.*`

## Maven Central Coordinates

The artifacts are published with the fork Maven group:

```text
io.github.duanluan.springframework:spring-framework-bom:5.3.42
io.github.duanluan.springframework:spring-core:5.3.42
io.github.duanluan.springframework:spring-context:5.3.42
io.github.duanluan.springframework:spring-web:5.3.42
io.github.duanluan.springframework:spring-webmvc:5.3.42
io.github.duanluan.springframework:spring-webflux:5.3.42
```

Example Maven BOM usage:

```xml
<dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>io.github.duanluan.springframework</groupId>
			<artifactId>spring-framework-bom</artifactId>
			<version>5.3.42</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
</dependencyManagement>
```

Example Gradle BOM usage:

```groovy
dependencies {
	implementation platform("io.github.duanluan.springframework:spring-framework-bom:5.3.42")
	implementation "io.github.duanluan.springframework:spring-webmvc"
}
```

If a project still receives `org.springframework:spring-*` through transitive
dependencies, add dependency substitution for the Spring modules used by that
project:

```groovy
configurations.configureEach {
	resolutionStrategy.dependencySubstitution {
		substitute module("org.springframework:spring-core") using module("io.github.duanluan.springframework:spring-core:5.3.42")
		substitute module("org.springframework:spring-context") using module("io.github.duanluan.springframework:spring-context:5.3.42")
		substitute module("org.springframework:spring-web") using module("io.github.duanluan.springframework:spring-web:5.3.42")
		substitute module("org.springframework:spring-webmvc") using module("io.github.duanluan.springframework:spring-webmvc:5.3.42")
		substitute module("org.springframework:spring-webflux") using module("io.github.duanluan.springframework:spring-webflux:5.3.42")
	}
}
```

## Security Backports

This branch applies selected fixes for the following Spring Framework advisories:

- CVE-2024-38819: Path traversal with static resources in WebMvc.fn and WebFlux.fn
- CVE-2024-38820: DataBinder case-insensitive binding issue caused by locale-specific case conversion
- CVE-2024-38828: DoS risk for Spring MVC controllers with `@RequestBody byte[]`

The public Spring Framework OSS repository does not provide an official
`v5.3.42` tag. This branch therefore does not claim to be byte-for-byte equal to
the commercial Spring Framework 5.3.42 build. It is based on `v5.3.39` and
backports the relevant public fix behavior for the advisories above.

## CVE-2024-38820 Fix

Official advisory:

- <https://spring.io/security/cve-2024-38820>

Public Spring Framework commit used as reference:

- <https://github.com/spring-projects/spring-framework/commit/23656aebc6c7d0f9faff1080981eb4d55eff296c>

Fix idea:

- `DataBinder` lowercases disallowed field names with `Locale.ROOT`.
- Field matching also lowercases candidate fields with `Locale.ROOT`.
- This avoids locale-specific conversions such as Turkish `i`/`I` behavior.

Files changed:

- `spring-context/src/main/java/org/springframework/validation/DataBinder.java`
- `spring-context/src/test/java/org/springframework/validation/DataBinderTests.java`

Regression test:

```bash
./gradlew :spring-context:test --tests org.springframework.validation.DataBinderTests.bindingWithDisallowedFieldsUsesLocaleIndependentLowerCase
```

## CVE-2024-38819 Fix

Official advisory:

- <https://spring.io/security/cve-2024-38819>

Public Spring Framework comparison used as reference:

- <https://github.com/spring-projects/spring-framework/compare/v6.1.13...v6.1.14>

Fix idea:

- Validate the original request path before static resource lookup.
- Validate percent-encoded input paths after decoding and path processing.
- Normalize duplicate slashes, backslashes, leading slashes, and encoded path segments before the final traversal check.
- Reject resolved resource paths that contain encoded traversal segments.
- Keep the resource inside the configured static resource location.

Files changed:

- `spring-webmvc/src/main/java/org/springframework/web/servlet/function/PathResourceLookupFunction.java`
- `spring-webmvc/src/test/java/org/springframework/web/servlet/function/PathResourceLookupFunctionTests.java`
- `spring-webflux/src/main/java/org/springframework/web/reactive/function/server/PathResourceLookupFunction.java`
- `spring-webflux/src/test/java/org/springframework/web/reactive/function/server/PathResourceLookupFunctionTests.java`

Regression tests:

```bash
./gradlew :spring-webmvc:test --tests org.springframework.web.servlet.function.PathResourceLookupFunctionTests.pathTraversalIsRejected
./gradlew :spring-webflux:test --tests org.springframework.web.reactive.function.server.PathResourceLookupFunctionTests.pathTraversalIsRejected
```

## CVE-2024-38828 Fix

Official advisory:

- <https://spring.io/security/cve-2024-38828>

Fix idea:

- `ByteArrayHttpMessageConverter` no longer pre-allocates a `ByteArrayOutputStream`
  from the request `Content-Length` header.
- The converter now reads the actual request body with Java 8 compatible stream
  copying.
- This avoids oversized or malformed `Content-Length` values causing excessive
  allocation, integer conversion issues, or request handling failures.

Files changed:

- `spring-web/src/main/java/org/springframework/http/converter/ByteArrayHttpMessageConverter.java`
- `spring-web/src/test/java/org/springframework/http/converter/ByteArrayHttpMessageConverterTests.java`

Regression test:

```bash
./gradlew -PmainToolchain=8 :spring-web:test --tests org.springframework.http.converter.ByteArrayHttpMessageConverterTests.readWithLargeContentLength
```

## Verification

The branch was verified with targeted regression tests and local publication:

```bash
./gradlew -PmainToolchain=8 :spring-web:test --tests org.springframework.http.converter.ByteArrayHttpMessageConverterTests.readWithLargeContentLength
./gradlew :spring-context:test --tests org.springframework.validation.DataBinderTests.bindingWithDisallowedFieldsUsesLocaleIndependentLowerCase
./gradlew :spring-webmvc:test --tests org.springframework.web.servlet.function.PathResourceLookupFunctionTests.pathTraversalIsRejected
./gradlew :spring-webflux:test --tests org.springframework.web.reactive.function.server.PathResourceLookupFunctionTests.pathTraversalIsRejected
./gradlew publishToMavenLocal
```

The current project integration also verified that these modules resolve to
`io.github.duanluan.springframework:5.3.42`:

- `spring-core`
- `spring-context`
- `spring-web`
- `spring-webmvc`
- `spring-webflux`

## Scanner Notes

This fork changes Maven coordinates. Some scanners decide findings by Maven
coordinates, some by SBOM package URL, some by class names, and some by CPE. If a
scanner cannot understand this self-maintained coordinate, attach this README,
the commit diff, and the regression test results as evidence.

This branch is focused on CVE-2024-38819, CVE-2024-38820, and CVE-2024-38828. It
does not remove deprecated Spring classes such as `HttpInvokerServiceExporter`;
scanners that report CVE-2016-1000027 by class presence may still flag
`spring-web`.

## License and Notice

The original Spring Framework license and notice files are retained:

- `LICENSE.txt`
- `NOTICE.txt`

Spring Framework is released under the Apache License, Version 2.0.
