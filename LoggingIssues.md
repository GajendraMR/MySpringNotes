# The Ask
With the Spring Cloud Gateway in picture, we had default logging (logback) provided by SpringBoot. We needed to add certain keys into the Logs so that it is picked up by Logz.io and we can filter the logs based on those key:values

# The Current Solution
Default Spring Logging in the Format
```
2024-06-12 11:03:33,048 [main] [INFO ] [org.example.logger.name]: logging-message
```

# The Options Available
## Option 1
Use the default logging and try to add in Key:Value so that it is picked up by Logz.io
### What was changed to achieve this?

### Key websites for more info

### Result
"DIDN'T WORK"

## Option 2
Use Log4j2 logging and try to add key:value so that it is picked up by Logz.io
### What was changed to achieve this?

### More Advanced on this option 👰‍♀️

### Key websites for more info

### Result
"TRYING IT OUT"

