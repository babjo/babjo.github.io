---
layout: post
title:  "스프링 시큐리티 튜토리얼 (Securing a Web Application)"
date:   2018-10-09 10:00:00 +0900
categories: learning-tech
tags: [spring, spring framework, 스프링, 스프링 프레임워크]
---

여전히 업무로 스프릥을 사용 중이나, 정확히 알고 쓰고 있는지는 의심이 들 때가 있어요. 그래서 그 의심을 해소하고자 스프링 레퍼런스를 펴놓고 다 읽기를 도전하는데요. 번번이 실패를 하죠. 그렇게 고민을 하며 시간을 보내던 찰나, 백기선님 유튜브 영상을 하나 보게 됩니다. 이름은 `[스프링 가이드] "토비의 스프링 그렇게 보지 마세요."`. 와우, 영상을 다 보니 결론이 레퍼런스를 모두 읽어나가는 일은 가상비가 나지 않는 일이라고 하시는군요. 필요한 기능들에 한하여 사용하는 방법을 알 수 있는 코드를 보는 정도로 시작하면서 업무에 적용하는 정도면 괜찮다고 하시더라구요. 공감하는 바이며 그래서 반대로 방법을 바꿨습니다. 레퍼런스가 아닌 튜토리얼 격파하기(?)로. 하나씩 살펴볼까해요. 그 중에 첫번째 스프링 시큐리티입니다.


우선 프로젝트 폴더를 만들고 구조를 잡아보아요. `src/main/java/hello`
```bash
~/IdeaProjects
❯ mkdir spring-security-tutorial

~/IdeaProjects
❯ cd spring-security-tutorial

~/IdeaProjects/spring-security-tutorial
❯ mkdir -p src/main/java/hello

~/IdeaProjects/spring-security-tutorial
❯ tree
.
└── src
    └── main
        └── java
            └── hello

4 directories, 0 files
```

그리고 `build.gradle`을 만들어줍시다.
```gradle
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:2.0.5.RELEASE")
    }
}

apply plugin: 'java'
apply plugin: 'eclipse'
apply plugin: 'idea'
apply plugin: 'org.springframework.boot'
apply plugin: 'io.spring.dependency-management'

bootJar {
    baseName = 'gs-securing-web'
    version =  '0.1.0'
}

repositories {
    mavenCentral()
}

sourceCompatibility = 1.8
targetCompatibility = 1.8

dependencies {
    compile("org.springframework.boot:spring-boot-starter-thymeleaf")
    compile("org.springframework.boot:spring-boot-starter-web")
    testCompile("junit:junit")
    testCompile("org.springframework.boot:spring-boot-starter-test")
    testCompile("org.springframework.security:spring-security-test")

}
```
잠깐 여기서 [spring-boot-gradle 플러그인](https://docs.spring.io/spring-boot/docs/current/gradle-plugin/reference/html) 기능들을 살펴보자면,
- classpath에 모든 jars를 모아서 `"über-jar"`를 만들어줍니다. (`"über-jar"` 안에 모든 jars 가 들어가 있어서 설치할 때 별도의 디펜던시에 고려가 필요 없어용)
- `public static void main()` 함수를 찾아줍니다.
- 플러그인 안에 내장된 디펜던시 리졸버로 스프링 부트 디펜던시에 버전을 맞춰줍니다.

다시 돌아와서 스프링 시큐리티를 적용하지 않은 페이지를 만들어봅시다. (물론 뒤에서 스프링 스큐리티 적용합니다.) 간단한 "Hello World" 페이지에요.

`src/main/resources/templates/home.html`
```html
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml" xmlns:th="http://www.thymeleaf.org" xmlns:sec="http://www.thymeleaf.org/thymeleaf-extras-springsecurity3">
    <head>
        <title>Spring Security Example</title>
    </head>
    <body>
        <h1>Welcome!</h1>

        <p>Click <a th:href="@{/hello}">here</a> to see a greeting.</p>
    </body>
</html>
```

보면 아시겠지만, 저 페이지에 `/hello`로 링크를 넣어줬어요. 관련해서 `hello.html`도 만들어보죠.

`src/main/resources/templates/hello.html`
```html
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml" xmlns:th="http://www.thymeleaf.org"
      xmlns:sec="http://www.thymeleaf.org/thymeleaf-extras-springsecurity3">
    <head>
        <title>Hello World!</title>
    </head>
    <body>
        <h1>Hello world!</h1>
    </body>
</html>
```

이 웹 어플리케이션은 당연히 스프링MVC 로 설정해서 이 template들을 웹 브라우저에서 볼 수 있도록 설정할거에요.

`src/main/java/hello/MvcConfig.java`
```java
package hello;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.ViewControllerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class MvcConfig implements WebMvcConfigurer {

    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/home").setViewName("home");
        registry.addViewController("/").setViewName("home");
        registry.addViewController("/hello").setViewName("hello");
        registry.addViewController("/login").setViewName("login");
    }

}
```

여기 `addViewController` 함수에서 4개 view controllers를 등록해요. 이중에 두개는 "home" (`home.html`)로 view를 정해주고 나머지 하나는 "hello" (`hello.html`)로 마지막 하나는 "login" (`login.html`)로 설정해줘요.

그리고 이 웹 어플리케이션을 실행할 `main()` 함수를 만들어아죠.

`src/main/java/hello/Application.java`
```java
package hello;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {

    public static void main(String[] args) throws Throwable {
        SpringApplication.run(Application.class, args);
    }

}
```

이렇게 실행하면, 아무 로그인 없이 `/`, `/home`, `/hello` 어디든 들어갈 수가 있어요. 이제 스프링 시큐리티를 본격적으로 달아볼 때이군요.

먼저 로그인을 하지 않으면, `/hello` 페이지로 못들어가게 만들어보죠. 자, 스프링 시큐리티 디펜던시를 추가하구요.

`build.gradle`
```gradle
dependencies {
    ...
    compile("org.springframework.boot:spring-boot-starter-security")
    ...
}
```

이렇게 바로 추가하고 나면 일단 모드 페이지에 대해서 모두 `basic` 인증이 들어가요. (`basic` 인증은 인증 방법 중에 하나에요. 스프링 시큐리티로 적용할 수도 있어요.) 그리고 인증된 유저만 `/hello`에 접근할 수 있도록 설정을 해보죠.

`src/main/java/hello/WebSecurityConfig.java`
```java
package hello;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;

@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .antMatchers("/", "/home").permitAll()
                .anyRequest().authenticated()
                .and()
            .formLogin()
                .loginPage("/login")
                .permitAll()
                .and()
            .logout()
                .permitAll();
    }

    @Bean
    @Override
    public UserDetailsService userDetailsService() {
        UserDetails user =
             User.withDefaultPasswordEncoder()
                .username("user")
                .password("password")
                .roles("USER")
                .build();

        return new InMemoryUserDetailsManager(user);
    }
}
```

`@EnableWebSecurity` 어노테이션을 달아주고 `WebSecurityConfigurerAdapter`을 상속받아 `configure(HttpSecurity)` 함수, `userDetailsService` 함수를 재정의 해줍니다. (보안 설정을 위해)

`configure(HttpSecurity)` 함수는 어떤 URL paths가 보호되어야하는지, 아니면 그럴 필요 없는지 정의해요. 우리는 `/`랑 `/home`는 로그인 없이 접근 가능도록 만들고 나머지 URL paths에 대해서는 모두 접근할 수 있도록 설정해요.

어떤 유저가 로그인이 필요하면 로그인 페이지로 이동을 해야하는데 이는 `loingPage()`로 설정할 수 있어요. 저희는 `/login`으로 가도록 했죠. 만약 여기서 로그인을 성공하면 로그인 페이지로 redirect 되기 전 페이지로 이동해요.

`userDetailsService()` 함수는 본래 DB에서 유저 정보를 조회하는 `UserDetailsService` bean을 만드는 함수인데, 저희는 간단하게 계정이 `"user"`이고 비밀번호가 `"password"`인 유저를 하나 만들어서 메모리에 저장하는 `InMemoryUserDetailsManager` 객체를 만들죠 (간단하게).

그리고 `/login` 페이지를 보여주기 위해, `loing.html` 파일을 만들어봅시다.

`src/main/resources/templates/login.html`
```html
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml" xmlns:th="http://www.thymeleaf.org"
      xmlns:sec="http://www.thymeleaf.org/thymeleaf-extras-springsecurity3">
    <head>
        <title>Spring Security Example </title>
    </head>
    <body>
        <div th:if="${param.error}">
            Invalid username and password.
        </div>
        <div th:if="${param.logout}">
            You have been logged out.
        </div>
        <form th:action="@{/login}" method="post">
            <div><label> User Name : <input type="text" name="username"/> </label></div>
            <div><label> Password: <input type="password" name="password"/> </label></div>
            <div><input type="submit" value="Sign In"/></div>
        </form>
    </body>
</html>
```

`login.html`에서 입력을 받는 두 `input` 태그에 각각 `username`이랑 `password`로 `name`을 지정해줬는데요. 이렇게 넣으면 사용자로부터 입력받은 값을 `/login`으로 POST 방식으로 전달해줘요. 그럼 스프링 시큐리티에서 제공하는 필터가 이 요청을 가로채서 인증을 하는거죠. 만약 유저가 인증에 실패하면 `/login?error`로 가고 `param.error`에 왜 실패했는지 값이 담겨져와요. (저희 페이지에서는 그 값이 있으면 화면에 `Invalid username and password.`라고 표시하도록 했어요.) 만약 로그아웃을 하면 `/login?logout`으로 가고 `param.logout`에 로그아웃했는지 값이 담겨져와요. (저희 페이지에서는 그 값이 있으면 `You have been logged out.`라고 표시하도록 했구요.)

지금은 로그아웃 페이지(`/logout`)로 이동할 수 있는 방법이 없으면 `hello.html`를 수정해서 로그아웃 페이지로 갈 수 있도록 만들어줍시다.

`src/main/resources/templates/hello.html`
```html
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml" xmlns:th="http://www.thymeleaf.org"
      xmlns:sec="http://www.thymeleaf.org/thymeleaf-extras-springsecurity3">
    <head>
        <title>Hello World!</title>
    </head>
    <body>
        <h1 th:inline="text">Hello [[${#httpServletRequest.remoteUser}]]!</h1>
        <form th:action="@{/logout}" method="post">
            <input type="submit" value="Sign Out"/>
        </form>
    </body>
</html>
```

참고로, `httpServletRequest#getRemoteUser()`로 화면에 로그인한 유저 이름을 출력하도록 했어요. 이제 `Sing Out` 버튼을 누르면 `/logout`으로 POST 요청을 보내고 성공하면 `/login?logout`으로 이동해요.

이제 여기까지하면 저희가 기본적으로 원했던 `/`, `/home`은 보안 없이 들어가고 보안이 필요한 경우 `/login`에서 로그인하고 `/hello` 페이지를 접근하는 것 그리고 로그아웃하는 것까지 모두 다 만들어봤어요. 여기까지입니다. 감사합니다.

완성된 코드는 여기에 올려놨어요:) https://github.com/babjo/spring-security-tutorial

#### 참고
- https://www.youtube.com/watch?v=97lYN9YW03Q
- https://spring.io/guides/gs/securing-web