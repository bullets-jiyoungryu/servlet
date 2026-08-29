# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

Spring MVC의 서블릿 기초를 학습하기 위한 Spring Boot 4.1.1 / Java 25 예제 프로젝트. `hello` 그룹, WAR 패키징, Gradle 9.7.1 래퍼.

## 명령어

```bash
./gradlew bootRun                                   # 앱 실행 (내장 톰캣, 기본 8080 포트)
./gradlew build                                     # 컴파일 + 테스트 + WAR 생성
./gradlew test                                      # 전체 테스트
./gradlew test --tests 'hello.servlet.ServletApplicationTests'          # 클래스 단위 실행
./gradlew test --tests '*ServletApplicationTests.contextLoads'          # 메서드 단위 실행
./gradlew bootWar                                   # build/libs/servlet-0.0.1-SNAPSHOT.war
```

동작 확인: 앱 실행 후 `curl 'http://localhost:8080/hello?username=kim'` → `hello kim`.

린터/포매터 태스크는 없다. `.idea/google-java-format.xml`이 있어 IntelliJ에서는 google-java-format이 걸려 있지만, Gradle 빌드에는 연결되어 있지 않다. 들여쓰기는 파일마다 다르므로 **수정 중인 파일의 기존 스타일을 그대로 따른다** (Spring Boot가 생성한 `ServletApplication`/`ServletInitializer`는 탭, 직접 작성한 `hello.servlet.basic`은 스페이스 4칸).

## 아키텍처

### 서블릿 등록 경로

이 프로젝트는 Spring MVC의 `@Controller`가 아니라 **표준 서블릿(`HttpServlet`) 직접 등록** 방식을 학습 대상으로 삼는다.

- `ServletApplication`에 붙은 `@ServletComponentScan`이 `@WebServlet` 애노테이션이 달린 클래스를 찾아 서블릿 컨테이너에 자동 등록한다.
- `HelloServlet`(`hello.servlet.basic`)처럼 `service()`를 오버라이드해 `HttpServletRequest`/`HttpServletResponse`를 직접 다루는 것이 이 리포의 기본 코드 형태다. 새 예제도 `hello.servlet.*` 하위에 만든다.
- 서블릿 API는 `jakarta.servlet.*` 네임스페이스를 쓴다 (`javax.servlet.*` 아님).
- `@ServletComponentScan`은 basePackages를 생략하면 애노테이션이 선언된 클래스의 패키지(= `hello.servlet`)부터 스캔한다. `@SpringBootApplication`의 컴포넌트 스캔도 동일 기준. 새 서블릿이 404가 나면 **패키지가 `hello.servlet` 밖인지부터 확인**하고, 밖에 둬야 한다면 `@ServletComponentScan("hello")`처럼 범위를 명시한다.

### Spring Boot 4 관련 주의점

Boot 3.x 예제/강의 코드를 그대로 옮기면 깨지는 지점이 있다.

- 스타터 이름이 바뀌었다: `spring-boot-starter-webmvc`(← `-web`), `spring-boot-starter-webmvc-test`(← `-test`), `spring-boot-starter-tomcat-runtime`(← `-tomcat`).
- `@ServletComponentScan`이 `org.springframework.boot.web.server.servlet.context` 패키지로 이동했다 (← `org.springframework.boot.web.servlet`). import를 옛 경로로 쓰면 컴파일이 안 된다.
- Lombok은 main/test 양쪽에 `compileOnly` + `annotationProcessor`로 설정되어 있으니 바로 쓸 수 있다.

### WAR 패키징 구성

`war` 플러그인 + `ServletInitializer`(`SpringBootServletInitializer` 상속) 조합으로 **외부 WAS 배포**도 가능하게 되어 있다. 톰캣이 `providedRuntime` 스코프이므로 WAR 안에 포함되지 않지만, `bootRun`/`bootWar`에서는 내장 톰캣이 정상 동작한다. `ServletInitializer`는 외부 톰캣 배포 시에만 쓰이는 진입점이므로, 로컬 실행 문제를 이 클래스에서 찾지 말 것.
