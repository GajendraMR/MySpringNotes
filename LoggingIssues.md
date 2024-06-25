# Technology
Spring Cloud Gateway

**Date**: 25/Jun/2024
**Author**: Gajendra Maddur Ramesh

# Requirement
Add custom fields (Key:Value) in the logs so that it is picked up by Logz.io and to use the fields to filter the logs.

# The Current Solution & Format
By default, Spring Cloud Gateway will be running Logback as its default logging module.

## Default Spring Logging Format
```
2024-06-12 11:03:33,048 [main] [INFO ] [org.example.logger.name]: i-am-a-long-message-logging-using-spring-default-format
```

## On Logz.io Format with Spring default enabled (disaplayed fewer fields)
| timestamp | container_name | container_id | log |
| --- | --- | --- | --- |
| 2024-06-12 11:03:33,048 | my_container_name | 1234567890example-12345 | [main] [INFO ] [org.example.logger.name]: i-am-a-long-message-logging-using-spring-default-format |

# The Solutions Available
## Option 1 [DIDN'T WORK]
Use the default logging and try to add in `Key:Value` so that it is picked up by Logz.io
### What was changed to achieve this?
The [Logging in Spring Boot](https://www.baeldung.com/spring-boot-logging) provides a detailed explaination on the type of loggings available and how to configure each logging.
With the help of the above page, I tried to configure the `logback-spring.xml` with below config to have the `app_name`, custom-correlation-id` & `device-type` fields present in logs.

```xml
<configuration>
  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <layout>
      <pattern>%d [%thread] %highlight(%-5level) app-name=${env:app_name:-mysampleapplication} CUSTOM-CORRELATION-ID=%X{custom-correlation-id} device-type=%X{device-type} %cyan(%logger{15}) : %kvp : %msg %ex %throwable%n</pattern>
    </layout>
  </appender>

  <root level="INFO">
    <appender-ref ref="CONSOLE" />
  </root>
</configuration>
```

In order to support the `MDC` [[What is MDC?](https://logback.qos.ch/manual/mdc.html)] or `%X` (conversion pattern to output the Thread Context Map or MDC), I tried using the MDC.put method to populate it in the code so that I can refer from the xml and log the details.
```
MDC.put("custom-correlation-id", "1234567890-12345");
MDC.put("device-type", "mobile");
```

### Result
I couldn't get it to work. Not sure if I had made a mis-configuration or if it doesn't work at all.


## Option 2 [SUCCESS]
Enable Log4j2 logging and try to add key:value so that it is picked up by Logz.io
### What was changed to achieve this?
#### Step 1:
Disable default Spring logging (logback) and add few dependencies in gradle file (take care of versions):
```
dependencies {
    implementation "org.apache.logging.log4j:log4j-layout-template-json:2.23.1"
    implementation "org.springframework.boot:spring-boot-starter-log4j2:3.3.6"
    implementation "org.apache.logging.log4j:log4j-slf4j-impl:2.23.1"
}

configurations {
    all {
        // Enabling Log4j2 requires disabling spring-boot-logging
        exclude group: 'org.springframework.boot', module: 'spring-boot-starter-logging'
    }
}

```

#### Step 2:
Again with reference to [Logging in Spring Boot](https://www.baeldung.com/spring-boot-logging) blog, I configured a file named `log4j2.xml` under `src/resources/` folder with below config
>  Note: The file named `log4j2-spring.xml` didn't work for me
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration>
  <SpringProfile name="local">
    <Appenders>
      <Console name="STDOUT" target="SYSTEM_OUT" follow="true">
        <PatternLayout
          pattern="%d [%t] [%highlight{%-5level}] %style{%logger{15}}{cyan} : %msg %ex %throwable%n" />
      </Console>
    </Appenders>
    <Loggers>
      <Root level="info" additivity="true">
        <AppenderRef ref="STDOUT" />
      </Root>
    </Loggers>
  </SpringProfile>

  <SpringProfile name="dev | qa">
    <Appenders>
      <Console name="STDOUT" target="SYSTEM_OUT" follow="true">
        <JsonTemplateLayout
          eventTemplateUri="classpath:LogstashJsonEventLayoutV1.json">
          <EventTemplateAdditionalField key="app_name"
            value="${env:app_name:-mysampleapplicationname}" />
          <EventTemplateAdditionalField
            key="env" value="${env:SPRING_PROFILES_ACTIVE}" />
        </JsonTemplateLayout>
      </Console>
    </Appenders>
    <Loggers>
      <Root level="info" additivity="true">
        <AppenderRef ref="STDOUT" />
      </Root>
    </Loggers>
  </SpringProfile>
</Configuration>
```
> [!TIP]
> You can see 2 `Appenders` blocks, one for `local` & the other for different `SPRING_PROFILES`. This is to have different patterns of logging as JSON pattern doesn't look good for troubleshooting locally.
>
> Also, I have used JSON Template Layout for more easier usage of log4j2 and adding up fields with the `EventTemplateAdditionalField`
> There are 4 available ready templates to choose from and can be found [here](https://github.com/vy/log4j2-logstash-layout/tree/master/layout/src/main/resources)
> The official documentation: [here](https://logging.apache.org/log4j/2.x/manual/json-template-layout.html)

#### Step 3:

And in the java file added below method call to populate other fields:
```
ThreadContext.put("custom-correlation-id", String.valueOf(exchange.getRequest().getHeaders().get("custom-correlation-id")));
ThreadContext.put("device-type", String.valueOf(exchange.getRequest().getHeaders().get("device-type")));
```

> [!TIP]
> The fields added using `ThreadContext` will be automatically logged into `mdc` key and looks like below in JSON output
> ```json
> mdc: {
>     "custom-correlation-id": "",
>     "device-type": ""
> }
> ```

#### The Output
1. Locally (it looks more beautiful than the below code block):
   ```
   2024-06-12 11:03:33,048 [main] [INFO ] [org.example.logger.name]: i-am-a-long-message-logging-using-spring-default-format
   ```
2. In Logz.io (displayed fewer fields):
   | timestamp | container_name | container_id | level | logger_name | message | mdc.custom-correlation-id | mdc.device-type | app_name |
   | --- | --- | --- | --- | --- | --- | --- | --- | --- |
   | 2024-06-12 11:03:33,048 | my_container_name | 1234567890example-12345 | [INFO ] | [org.example.logger.name] | i-am-a-long-message-logging-using-spring-default-format | 123456789-12345 | mobile | mysampleapplicationname |

### More Advanced on this option 👰‍♀️
Unit testing was harder with the log4j2 implementation, only because I was not from Java background and logback supported `iLoggingEvent` & `ListAppender` for testing the log messages.
After hours of searching over the stack-overflow. Found this [amazing comment](https://stackoverflow.com/a/28682591) which got the Unit Test running effectively. All the best :) 

### Result
IT WORKED 🚀

## Many Thanks
- google.co.uk
- https://www.baeldung.com/
- https://logging.apache.org/
- https://github.com/
- https://logback.qos.ch/
- https://stackoverflow.com/
- All the bloggers
- All the commenters on Stack Overflow
- My team for asking me to work on this
