# CURL Logger

[![Build Status](https://travis-ci.org/dzieciou/curl-logger.svg?branch=master)](https://travis-ci.org/dzieciou/curl-logger/)
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/com.github.dzieciou.testing/curl-logger/badge.svg)](https://maven-badges.herokuapp.com/maven-central/com.github.dzieciou.testing/curl-logger)

Logs HTTP requests sent by [REST-assured][9] as [CURL][1] commands.

The following request from REST-assured test
```java  
given()
  .config(config)
  .redirects().follow(false)
.when()
  .get("http://google.com")
.then()
  .statusCode(302); 
```
will be logged as:
```ba
curl 'http://google.com/' -H 'Accept: */*' -H 'Content-Length: 0'  -H 'Connection: Keep-Alive' 
  -H 'User-Agent: Apache-HttpClient/4.5.2 (Java/1.8.0_112)' -H 'Content-Type: multipart/mixed' 
  --compressed -k -v
```

This way testers and developers can quickly reproduce an issue and isolate its root cause. 

## Usage

Latest release:

```xml
<dependency>
  <groupId>com.github.dzieciou.testing</groupId>
  <artifactId>curl-logger</artifactId>
  <version>1.0.5</version>
</dependency>
```
   
### Using with REST-assured client 
    
When sending HTTP Request with REST-assured, you must create `RestAssuredConfig` instance first
 as follows:
        
```java
RestAssuredConfig config = CurlRestAssuredConfigFactory.createConfig();  
```
  
and then use it:

```java
given()
  .config(config)
  ...
```

If you already have a `RestAssuredConfig` instance, you may reconfigure it as follows:

```java
RestAssuredConfig config = ...;
config = CurlRestAssuredConfigFactory.updateConfig(config);  
```

The library provides a number of options for the way curl is generated and logged. They can be
defined with `Options` class. For instance:
 
```java
Options options = Options.builder()...build();
RestAssuredConfig config = CurlRestAssuredConfigFactory.createConfig(options);  
```

There is a separate section listing all options.
 
### Configuring logger 

CURL commands are logged to a "curl" logger. The library requires only the logger to be [slf4j][4]-compliant, e.g.,
using [logback][5]. Sample logback configuration that logs all CURL commands to standard system output would be:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%-4relative [%thread] %-5level %logger{35} - %msg %n</pattern>
        </encoder>
    </appender>
    <logger name="curl" level="DEBUG">
        <appender-ref ref="STDOUT"/>
    </logger>
</configuration>
```


## Options

### Printing stacktrace

The library provides a way to log stacktrace where the curl was generated:
```java  
Options.builder().logStacktrace().build();
```
This might be particularly useful when your test is sending multiple requests and you cannot find
which request generated  printed curl command.

By default `CurlRestAssuredConfigFactory#createConfig` creates configuration that prints a curl command without stacktrace.

### Configure log level

There is a way to define at which log level the log statement should be created:
```java  
Options.builder().logLevel(Level.INFO).build();
```

By default the curl commands are generated at `DEBUG` level, but depending on the context it might 
be handy to log at a higher level.


### Generating curl for Windows vs Unix 

The curl command generated by the library is platform-specific.

For instance, curl command generated for Unix platform

```bash
curl 'http://testapi.com/post' --data-binary $'{\r\n   \'name\':\'Administra\xe7\xe3o\',\r\n   
  \'email\':\'admin\x40gmail.com\',\r\n   \'password\':\'abc%"\'\r\n}'
```

will look different when generated for Windows platform:

```cmd
curl "http://testapi.com/post" --data-binary "{"^

"   'name':'Administração',"^

"   'email':'admin@gmail.com',"^

"   'password':'abc"%"""'"^

"}"
```

The difference is in:
- how strings are quoted,
- how different characters (non-ASCII characters,
non-printable ASCII characters and characters with special meaning) are escaped,
- and how new lines in multiline command are printed.

By default, the library assumes the target platform on which curl command will be executed is the 
same as the platform where it was generated. However, when you work on two different platforms, 
sometimes a curl command generated on Unix might not be portable to Windows, and vice-versa.  

In such situations, the library provides a way to force target platform, e.g.,:
```java
Options.builder().targetPlatform(Platform.UNIX).build();
```

Even if origin and target platform are the same, some characters must be escaped, because 
the generated curl command will be copy-pasted on a multitude of setups by a wide range of people. 
I don't want the result to vary based on people's local encoding or misconfigured terminals. The 
downside of this might be less legible commands like `Administra\xe7\xe3o` instead of 
`Administração`. If you want your commands to be more legible (and bound to specific terminal setups)
you may disable escaping non-ASCII characters by using the following option (for Unix only):
```java
Options.builder().dontEscapeNonAscii().build();
```

### Printing curl in multiple lines

The library enables printing a curl command in multiple lines:
```java
Options.builder().printMultiliner().build();
```

On Unix systems it will look like this:
```bash
curl 'http://google.pl/' \ 
  -H 'Content-Type: application/x-www-form-urlencoded' \ 
  --data-binary 'param1=param1_value&param2=param2_value' \ 
  --compressed \ 
  -k \ 
  -v
```
and on Windows:
```cmd
curl 'http://google.pl/' ^ 
  -H 'Content-Type: application/x-www-form-urlencoded' ^ 
  --data-binary 'param1=param1_value&param2=param2_value' ^ 
  --compressed ^ 
  -k ^ 
  -v
```

### Logging request body

Note, for either platform, the body of a request is always logged as `--binary-data` instead of 
`--data` because the latter  strips newline (`\n`) and carriage return (`\r`) characters. 

### Printing curl parameters in long form

The library enables printing longer form of curl parameters, e.g. `--header` instead of `-H`:
```java
Options.builder().useLongForm().build();
```

Here's an example of a curl command generated with parameters in default short form (note, not 
all parameters have corresponding short form):

```bash
curl 'http://google.pl/' -H 'Content-Type: application/x-www-form-urlencoded' -H 'Host: google.pl' 
  -H 'User-Agent: 'User-Agent: Apache-HttpClient/4.5.2 (Java/1.8.0_112)' 
  -H 'Connection: Keep-Alive' --data-binary 'param1=param1_value&param2=param2_value' --compressed -k -v
```

After enabling long form option it would look as follows:

```bash
curl 'http://google.pl/' -header 'Content-Type: application/x-www-form-urlencoded' 
  --header 'Host: google.pl'  --header 'User-Agent: 'User-Agent: Apache-HttpClient/4.5.2 
     (Java/1.8.0_112)' --header 'Connection: Keep-Alive' 
  --data-binary 'param1=param1_value&param2=param2_value' --compressed --insecure --verbose
```

By default `CurlRestAssuredConfigFactory#createConfig` create configuration  that prints
 a curl command parameters in short form.

## Updating curl command before print

The library provides a way to modify curl command before 
printing:
```java
Options.builder().updateCurl(curl -> ...).build();
```

`#updateCurl` method takes instance of `Consumer<CurlCommand>` class. `CurlCommand` is a mutable 
representation of curl and offers a number of methods to modify it: `#addHeader`, `#removeHeader`, etc.

This is useful to:
* modify generated curl to test different variations of the same case
* remove certain headers or parameters to make curl command more consise and thus easier to read

For instance, if you would like skip common headers like "Host", "User-Agent" and "Connection" 
from the following curl command:
```bash
curl 'http://google.pl/' -H 'Content-Type: application/x-www-form-urlencoded' -H 'Host: google.pl'  
  -H 'User-Agent: 'User-Agent: Apache-HttpClient/4.5.2 (Java/1.8.0_112)' 
  -H 'Connection: Keep-Alive'  --data-binary 'param1=param1_value&param2=param2_value' --compressed -k -v
```

you should define `Options` as follows:
```java
Options.builder()
  .updateCurl(curl -> curl
    .removeHeader("Host")
    .removeHeader("User-Agent")
    .removeHeader("Connection"))
  .build();
```

As a result it will generate the following curl:
```
curl 'http://google.pl/' -H 'Content-Type: application/x-www-form-urlencoded' 
  --data-binary 'param1=param1_value&param2=param2_value' --compressed -k -v
```



## Other features

### Logging attached files

When you attach a file to your requests via `multiPart`, e.g., sending content of "README.md" file

```java
given()
  .config(CurlRestAssuredConfigFactory.createConfig())
  .baseUri("http://someHost.com")
  .multiPart("myfile", new File("README.md"), "application/json")
.when()
  .post("/uploadFile");
```

the library will automatically include reference to it instead of pasting its content:
```
curl 'http://somehost.com/uploadFile' -F 'myfile=@README.md;type=application/json' -X POST ...
```

Note, this won't work when uploading File not via `multipart`. Instead, content of the file will be
printed.

```java
given()
  .config(CurlRestAssuredConfigFactory.createConfig())
  .baseUri("http://someHost.com")
  .body(new File("README.md"))
.when()
  .post("/uploadFile");
```
### Custom curl handling

By default generated curls are logged. However, there's a way to process curl by one or more custom
handlers by providing a list of custom handlers, implementing `CurlHandler` interface. For instance, 
to store generated curls in a variable one could write:

```java
final List<String> curls = new ArrayList<>();
CurlHandler handler = new CurlHandler() {
  @Override
  public void handle(String curl, Options options) {
    curls.add(curl);
  }
};
List<CurlHandler> handlers = Arrays.asList(handler);
CurlRestAssuredConfigFactory.createConfig(handlers)
```

## Prerequisities

* JDK 8
* Dependencies with which I tested the solution

```xml
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>4.5.3</version>
</dependency>
<dependency>
    <groupId>io.restassured</groupId>
    <artifactId>rest-assured</artifactId>
    <version>4.3.1</version>
</dependency>
```

## Releases

2.0.0:
* Tested with latest REST-assured (4.3.1)
* Support for custom log levels (thanks to Jérémie Bresson for pull request)
* Support for custom curl handlers 
* Backward-incompatible change: `CurlLoggingRestAssuredConfigFactory` renamed to 
`CurlRestAssuredConfigFactory`. 

1.0.5:
* Upgrade to REST-assured 4.0.0.
* Update test scope dependencies.
* Fix character escaping for both POSIX and Windows platforms (many thanks to Chirag008 for rausung
the issue: https://github.com/dzieciou/curl-logger/issues/25)

1.0.4:
* Bug fix: HTTPS protocol was not always recognized correctly 
(https://github.com/dzieciou/curl-logger/issues/17). Many thanks to pafitchett-ks for troubleshooting.
* Support slf4j 1.8.0-beta2.
* Support rest-assured 3.2.0. 

1.0.3:
* Bug fix: Invalid basic authentication headers are failing curl generation 
(https://github.com/dzieciou/curl-logger/issues/15)

1.0.2:
* Bug fix: CurlLogger was failing when multiple Cookie headers are present in HTTP Request. Now it
only prints warning (https://github.com/dzieciou/curl-logger/issues/13)

1.0.1:
* Bug fix: `CurlLoggingRestAssuredConfigBuilder` was not updating `RestAssuredConfig` properly 
(https://github.com/dzieciou/curl-logger/issues/4): 

1.0.0:

* First major release with stable public API
* Provided a way to force target platform of generated curl command
* Backward-incompatible change: `CurlLoggingRestAssuredConfigBuilder` replaced with 
`CurlRestAssuredConfigFactory` that uses `Options` class to configure curl generation process.

0.7:

* Added possibility to print shorter versions of curl parameters, e.g., -v instead of --verbose
* Added possibility to modify a curl command before printing it, inspired by the suggestion from 
Alexey Dushen (blacky0x0): https://github.com/dzieciou/curl-logger/issues/2.

0.6:
* Fixed bug: For each cookie a separate `-b cookie=content` parameter was generated (https://github.com/dzieciou/curl-logger/issues/4)
* Upgraded to REST-assured 3.0.2
* Simplified curl-logger configuration with `CurlLoggingRestAssuredConfigBuilder`, based on suggestion from Tao Zhang (https://github.com/dzieciou/curl-logger/issues/4)

0.5:

* Upgraded to REST-assured 3.0.1 that contains important fix impacting curl-logger: Cookie attributes are no longer sent in request in accordance with RFC6265. 
* Fixed bug: cookie values can have = sign inside so we need to get around them somehow
* Cookie strings are now escaped
* `CurlLoggingInterceptor`'s constructor is now protected to make extending it possible 
* `CurlLoggingInterceptor` can now be configured to print a curl command in multiple lines
 

0.4:
 
* Upgraded to REST-assured 3.0

0.3:

 * Each cookie is now defined with `-b` option instead of `-H`
 * Removed heavy dependencies like Guava
 * Libraries like REST-assured and Apache must be now provided by the user (didn't want to constrain users to a specific version)
 * Can log stacktrace where curl generation was requested

0.2:

 * Support for multipart/mixed and multipart/form content types
 * Now all generated curl commands are `--insecure --verbose`
 
0.1:

 * Support for logging basic operations



## Similar tools
  
* Chrome Web browser team has ["Copy as CURL"][7] in the network panel, similarly [Firebug add-on][8] for Firefox.
* OkHttp client provides similar request [interceptor][3] to log HTTP requests as curl command. 
* [Postman add-on][6] for Chrome provides a way to convert prepared requests as curl commands.


  [1]: https://curl.haxx.se/
  [2]: https://github.com/dzieciou/curl-logger/issues
  [3]: https://github.com/mrmike/Ok2Curl 
  [4]: http://www.slf4j.org/
  [5]: http://logback.qos.ch/
  [6]: https://www.getpostman.com/docs/creating_curl
  [7]: https://coderwall.com/p/-fdgoq/chrome-developer-tools-adds-copy-as-curl
  [8]: http://www.softwareishard.com/blog/planet-mozilla/firebug-tip-resend-http-request/
  [9]: http://rest-assured.io/

## Bugs and features request

Report or request in [Issues][2].

## Contribute!

This is an open-source library, and contributions are welcome. You're welcome to fork this project 
and send me a pull request.

## Supporting 

This is an open-source library that I give for free to the community as my way of saying thank you
for all the support I have received from this community.

If you like my work and want to say thank you as well then:

[![Buy me a coffee](https://cdn.buymeacoffee.com/buttons/default-blue.png)](https://www.buymeacoffee.com/dzieciou)


