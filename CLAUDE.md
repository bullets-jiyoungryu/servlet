# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

서블릿부터 스프링 MVC까지 단계적으로 리팩터링해 가는 학습용 Spring Boot 4.1.1 / Java 25 예제 프로젝트. `hello` 그룹, WAR 패키징, Gradle 9.7.1 래퍼.

각 단계의 코드를 **버전 패키지로 모두 보존**하는 것이 이 리포의 성격이다. 앞 단계 코드를 지우거나 리팩터링하지 않는다.

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

`springmvc` 단계 이전의 코드는 스프링 빈이 아니라 **표준 서블릿(`HttpServlet`) 직접 등록**으로 동작한다.

- `ServletApplication`에 붙은 `@ServletComponentScan`이 `@WebServlet` 애노테이션이 달린 클래스를 찾아 서블릿 컨테이너에 자동 등록한다.
- `HelloServlet`(`hello.servlet.basic`)처럼 `service()`를 오버라이드해 `HttpServletRequest`/`HttpServletResponse`를 직접 다루는 것이 `frontcontroller` 단계까지의 기본 코드 형태다.
- 서블릿 API는 `jakarta.servlet.*` 네임스페이스를 쓴다 (`javax.servlet.*` 아님).
- `@ServletComponentScan`은 basePackages를 생략하면 애노테이션이 선언된 클래스의 패키지(= `hello.servlet`)부터 스캔한다. `@SpringBootApplication`의 컴포넌트 스캔도 동일 기준. 새 서블릿이 404가 나면 **패키지가 `hello.servlet` 밖인지부터 확인**하고, 밖에 둬야 한다면 `@ServletComponentScan("hello")`처럼 범위를 명시한다.

### 학습 단계별 패키지 진행

`hello.servlet.web` 하위는 "서블릿만 쓰던 코드 → 스프링 MVC"로 가는 리팩터링 단계를 버전 패키지로 보존한 것이다. 모든 단계가 `index.html`에서 동시에 접근 가능해야 한다.

| 패키지 | URL | 핵심 |
|---|---|---|
| `web.servlet` | `/servlet/members/*` | 서블릿에서 HTML을 직접 write |
| `web.servletmvc` | `/servlet-mvc/members/*` | `RequestDispatcher.forward()`로 JSP에 위임 |
| `web.frontcontroller.v1` | `/front-controller/v1/*` | 프론트 컨트롤러 도입, `ControllerV1` |
| `...v2` | `/front-controller/v2/*` | `MyView` 반환으로 forward 중복 제거 |
| `...v3` | `/front-controller/v3/*` | `ModelView` + `paramMap`, 컨트롤러에서 서블릿 종속 제거 |
| `...v4` | `/front-controller/v4/*` | viewName(String) 직접 반환 |
| `...v5` | `/front-controller/v5/v3/*`, `/v5/v4/*` | `MyHandlerAdapter`로 v3·v4 컨트롤러 동시 지원 |
| `web.springmvc.old` | `/springmvc/old-controller`, `/springmvc/request-handler` | v5 구조에 대응하는 스프링 실제 구현 확인용 |
| `web.springmvc.v1` | `/springmvc/v1/members/*` | `@Controller` + `@RequestMapping`, `ModelAndView` |
| `...v2` | `/springmvc/v2/members/*` | 클래스 레벨 `@RequestMapping`으로 컨트롤러 통합 |
| `...v3` | `/springmvc/v3/members/*` | `@GetMapping`/`@PostMapping`, `@RequestParam`, `Model`, String 반환 |

`v5`의 `handlerMappingMap` / `handlerAdapters`는 스프링의 `HandlerMapping` / `HandlerAdapter`에 그대로 대응한다. `springmvc.old`는 그 대응을 눈으로 확인하는 예제다. `@Component("/springmvc/old-controller")`처럼 **빈 이름을 `/`로 시작**시키면 `BeanNameUrlHandlerMapping`이 그 이름을 URL로 등록한다. 빈 이름 = URL이므로 중복되면 404가 아니라 `ConflictingBeanDefinitionException`으로 기동 자체가 실패한다.

### 뷰 해석 경로가 두 갈래다

`/WEB-INF/views/` 아래 JSP 3개(`new-form`, `save-result`, `members`)를 모든 단계가 공유하는데, 경로를 만드는 주체가 다르다.

- `frontcontroller` 계열 — `FrontControllerServletV5.viewResolver()` 등 **자바 코드에 하드코딩** (`"/WEB-INF/views/" + viewName + ".jsp"`). `MyView`가 `forward`한다.
- `springmvc` 계열 — `application.properties`의 `spring.mvc.view.prefix=/WEB-INF/views/`, `spring.mvc.view.suffix=.jsp`

JSP를 옮기면 **양쪽 다** 고쳐야 한다. properties만 바꾸면 frontcontroller 단계가 조용히 404가 난다.

### 도메인

`MemberRepository`는 `private` 생성자 + `static` 필드 기반 싱글턴이라 **모든 예제가 같은 저장소를 공유**한다. 동시성 처리는 일부러 넣지 않았다. 테스트에서는 `@AfterEach`로 `clearStore()`를 호출하지 않으면 상태가 샌다 (`MemberRepositoryTest` 참고).

### Spring Boot 4 관련 주의점

Boot 3.x 예제/강의 코드를 그대로 옮기면 깨지는 지점이 있다.

- 스타터 이름이 바뀌었다: `spring-boot-starter-webmvc`(← `-web`), `spring-boot-starter-webmvc-test`(← `-test`), `spring-boot-starter-tomcat-runtime`(← `-tomcat`).
- `@ServletComponentScan`이 `org.springframework.boot.web.server.servlet.context` 패키지로 이동했다 (← `org.springframework.boot.web.servlet`). import를 옛 경로로 쓰면 컴파일이 안 된다.
- Jackson이 3.x다. `ObjectMapper`는 `tools.jackson.databind`에서 import 한다 (← `com.fasterxml.jackson.databind`).
- JSP 의존성은 이미 `build.gradle`에 있다 (`tomcat-embed-jasper`, `jakarta.servlet.jsp.jstl-api`, `org.glassfish.web:jakarta.servlet.jsp.jstl`). 강의의 `javax.servlet:jstl`로 되돌리지 말 것 — Boot 4 BOM 관리 대상이 아니라 버전이 해석되지 않고 `Could not find javax.servlet:jstl:`로 `compileClasspath` 구성부터 실패한다. JSP의 taglib URI는 `http://java.sun.com/jsp/jstl/core`를 그대로 쓴다.
- Lombok은 main/test 양쪽에 `compileOnly` + `annotationProcessor`로 설정되어 있으니 바로 쓸 수 있다.

### WAR 패키징 구성

`war` 플러그인 + `ServletInitializer`(`SpringBootServletInitializer` 상속) 조합으로 **외부 WAS 배포**도 가능하게 되어 있다. 톰캣이 `providedRuntime` 스코프이므로 WAR 안에 포함되지 않지만, `bootRun`/`bootWar`에서는 내장 톰캣이 정상 동작한다. `ServletInitializer`는 외부 톰캣 배포 시에만 쓰이는 진입점이므로, 로컬 실행 문제를 이 클래스에서 찾지 말 것.

## 새 예제 추가 시

1. `hello.servlet.web.<단계>` 하위에 만든다 (`hello.servlet` 밖이면 스캔되지 않는다).
2. `src/main/webapp/index.html`의 목차에 링크를 추가한다 — 여기가 사실상 라우팅 인덱스다.
3. 앞 버전 패키지는 건드리지 않는다.
4. 커밋 메시지는 이 리포의 관례상 `feat: <강의 번호>. <강의 제목>` 형식이다 (예: `feat: 43. 스프링 MVC - 실용적인 방식`).
