---
layout: post
title: Spring Security and Session with Redis
date: '2017-10-30'
summary: "In this article I will show how to externalize users' session information from application containers and store it on Redis using Spring projects."
---

There was this product I was working on with my team where we wanted to achieve a multi-instance production environment. This would bring us several benefits like zero-downtime deploys, greater availability assurance and scalability in order to attend high demand peaks.

We had two main problems to solve in order to achieve this: store users' session information outside the application container and externalize the websockets host.

This is the first part on this report on how to externalize application container state where we solve the first of the two problems. The source for this demo is available on [Github](https://github.com/selzlein/spring-session-redis-demo).

## Scenario

The product in this matter is a web application built on [Spring Framework](https://spring.io/) and a few of its sub-projects: [Data](http://projects.spring.io/spring-data/) and [Security](http://projects.spring.io/spring-security/).

## Problem

Users' session information were currently kept in-memory by our application container, which  was [Wildfly](http://wildfly.org/). As it seems, it is a problem when running on a multi-instance environment to have sessions stored in every server. That may cause unexpected application behavior when one client request hits server A but a second gets handled by server B, where this user's session does not exist.

Another problem would be when server A goes down and from then on, server B has to handle all the load. That would occur ok if all sessions on server A were not lost.

## Solution

Store all sessions in the same place, where they would be available and manageable for all running application container instances. This way when one server goes down the others know where to find session information that the dead server was working with.

Spring framework has one project under its umbrella which provides integration with [Redis](https://redis.io/), which in summary is a in-memory key-value database. This sub-project is called [Spring Session](https://projects.spring.io/spring-session/) and implements several features, one among them is abstracting `HttpSession` to not depend on application container specific solution.

### Spring Session with Redis

The first step is to add Spring Session Data Redis and [Lettuce](https://mvnrepository.com/artifact/biz.paluch.redis/lettuce) dependencies:

{% highlight xml %}
<dependency>
	<groupId>org.springframework.session</groupId>
	<artifactId>spring-session-data-redis</artifactId>
	<version>1.3.1.RELEASE</version>
</dependency>
<dependency>
	<groupId>biz.paluch.redis</groupId>
	<artifactId>lettuce</artifactId>
	<version>4.4.1.Final</version>
</dependency>
{% endhighlight %}

Having the dependencies ok, it is time to configure the connection factory and enable Spring Session with Redis. This is done by the following config class:

{% highlight java %}
@EnableRedisHttpSession
public class Config {

  @Bean
  public LettuceConnectionFactory connectionFactory() {
    return new LettuceConnectionFactory();
  }

}
{% endhighlight %}

[@EnableRedisHttpSession](https://docs.spring.io/spring-session/docs/current/api/org/springframework/session/data/redis/config/annotation/web/http/EnableRedisHttpSession.html) will allow `HttpSession` to be backed by Spring Session and therefore stored in Redis.

The `LettuceConnectionFactory` bean will be the connection factory Spring Session requires. It is in here that host and port configuration for Redis can be specified. Default values for these are `localhost` and `6379` respectively.

This is all what it takes to have Spring Session enabled and integrated to Redis.

### Spring Security

In order to demo user session stored and retrieved from Redis, let's configure Spring Security. First of all, we add Spring Security dependency:

{% highlight xml %}
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId>
</dependency>
{% endhighlight %}

Having that done we configure Spring Security through the following class:

{% highlight java %}
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http
      .authorizeRequests()
        .anyRequest().authenticated()
      .and()
        .formLogin();
  }

  @Autowired
  public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
    auth.inMemoryAuthentication()
      .withUser("admin").password("admin").roles("ADMIN");
  }

}
{% endhighlight %}

`configure` is simply making sure that any request has a user authenticated.
`configureGlobal` defines an in-memory authentication with a *admin/admin* dumb user.

That's it on the configuration part. Now, let's play and see it working.

### Managing Sessions

The following controller defines an endpoint that counts page views per session and stores the result in the session itself:

{% highlight java %}
@Controller
public class IndexController {

  @GetMapping("/")
  public String index(HttpServletRequest request, Model model) {
    Integer pageViews = 1;
    if (request.getSession().getAttribute("pageViews") != null) {
      pageViews += (Integer) request.getSession().getAttribute("pageViews");
    }
    request.getSession().setAttribute("pageViews", pageViews);
    model.addAttribute("pageViews", pageViews);
    return "index";
  }

}
{% endhighlight %}

And this is its respective [Thymeleaf](http://www.thymeleaf.org/) template:

{% highlight html %}
<!DOCTYPE HTML>
<html xmlns:th="http://www.thymeleaf.org">
<head>
<meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
<title>Spring Session Redis Demo</title>
</head>
<body>
  <p th:text="'Page views: ' + ${pageViews}">Page views here</p>

  <form th:action="@{/logout}" method="post">
    <input type="submit" value="Logout" />
  </form>
</body>
</html>
{% endhighlight %}

This template renders the number of page views which is populated in our controller. It also presents a form that allows users to logout.

### Playing the result

Start our demo application and go to `localhost:8080`. Since we configured Spring Security to make sure any request is authenticated, the first page to be presented is `login`:

![Login page](/img/spring-security-session-redis/login.png "Login page")

Log in as *admin/admin*. `IndexController` will be called and the session attribute `pageViews` is created and populated in the user session:

![Index page](/img/spring-security-session-redis/index.png "Index page")

Every time you refresh this page the `pageViews` session attribute gets incremented and updated in the session which in turn is stored in Redis. If user logs out and log in again a new session is created and the counter will start over.

Now the true gain we have with this environment is that if we stop our application as start again user session will be intact. When the user hits refresh he will still be logged in and as one would expect, his `pageViews` attribute will continue to be incremented from where it was before the application was restarted.

## Conclusion

In this article we saw how to configure Spring Session with Redis and have Spring Security integrated in this environment. All this allowed us to store users' sessions in Redis. Therefore, they are not lost when an application container goes down.

Another benefit is that when using a centralized session storage it does not matter which application container handles a client request because any of them have accessed to session information.

I hope it helps and feedback is always welcome! :)