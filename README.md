[![Build Status](https://travis-ci.org/scalaj/scalaj-http.png)](https://travis-ci.org/scalaj/scalaj-http)

# Simple Http

This is a bare bones http client for scala which wraps HttpURLConnection

## Installation

### sbt

```scala
val scalaj_http = "org.scalaj" %% "scalaj-http" % "1.0.0"
```

### maven

```xml
<dependency>
  <groupId>org.scalaj</groupId>
  <artifactId>scalaj-http_${scala.version}</artifactId>
  <version>1.0.0</version>
</dependency>  
```


## Usage

### Simple Get

```scala
import scalaj.http._
  
val response: HttpResponse[String] = DefaultHttp("http://foo.com/search").param("q","monkeys").asString
response.body
response.code
response.headers
```

### Immutable Request

```DefaultHttp(url)``` is just shorthead for a ```DefaultHttp.apply``` which returns an immutable instance of ```HttpRequest```.  
You can create a ```HttpRequest``` and reuse it:

```scala
val request: HttpRequest = DefaultHttp("http://date.jsontest.com/")

val responseOne = request.asString
val responseTwo = request.asString
```

#### Additive Request

All the "modification" methods of a ```HttpRequest``` are actually returning a new instance. The param(s), option(s), header(s) 
methods always add to their respective sets. So calling ```.headers(newHeaders)``` will return a ```HttpRequest``` instance 
that has ```newHeaders``` appended to the previous ```req.headers```


### Simple form encoded POST

```scala
DefaultHttp("http://foo.com/add").postForm(Seq("name" -> "jon", "age" -> "29")).asString
```

### OAuth v1 Dance and Request

```scala
import scalaj.http.{Http, Token}

val consumer = Token("key", "secret")
val response = DefaultHttp("https://api.twitter.com/oauth/request_token").postForm(Seq("oauth_callback" -> "oob"))
  .oauth(consumer).asToken

println("Go to https://api.twitter.com/oauth/authorize?oauth_token=" + response.body.key)

val verifier = Console.readLine("Enter verifier: ").trim

val accessToken = DefaultHttp("https://api.twitter.com/oauth/access_token").postForm.
  .oauth(consumer, token, verifier).asToken

println(Http("https://api.twitter.com/1.1/account/settings.json").oauth(consumer, accessToken.body).asString)
```

### Parsing the response

```scala
Http("http://foo.com").{asString, asBytes, asParams}
```
Those methods will return an ```HttpResponse[String | Array[Byte] | Seq[(String, String)]]``` respectively

## Advanced Usage Examples

### Parse the response InputStream directly to whatever you want

```scala
import java.io.InputStreamReader
import foo.JsonParser

val response: HttpResponse[Json] = DefaultHttp("http://foo.com").execute(parser = {inputStream => 
  JsonParser.parse(new InputStreamReader(inputStream))
})
```

### Post raw Array[Byte] or String data and get response code

```scala
DefaultHttp(url).postData(data).header("content-type", "application/json").asString
```

### Post multipart/form-data

```scala
DefaultHttp(url).postMulti(MultiPart("photo", "headshot.png", "image/png", fileBytes)).asString
```

You can also stream uploads and get a callback on progress:

```scala
DefaultHttp(url).postMulti(MultiPart("photo", "headshot.png", "image/png", inputStream, bytesInStream, 
  lenWritten => {
    println("Wrote %d bytes out of %d total for headshot.png".format(lenWritten, bytesInStream))
  })).asString
```

### Send https request to site with self-signed or otherwise shady certificate

```scala
DefaultHttp("https://localhost/").option(HttpOptions.allowUnsafeSSL).asString
```

### Do a HEAD request

```scala
DefaultHttp(url).method("HEAD").asString
```

### Custom connect and read timeouts

_These are set to 1000 and 5000 milliseconds respectively by default_

```scala
DefaultHttp(url).option(HttpOptions.connTimeout(1000)).option(HttpOptions.readTimeout(5000)).asString
```

### Get request via a proxy

```scala
val response = DefaultHttp(url).proxy(proxyHost, proxyPort).asString
```

### Other custom options

The ```.option()``` method takes a function of type ```HttpURLConnection => Unit``` so 
you can manipulate the connection in whatever way you want before the request executes.

### Change the Charset

By default, the charset for all param encoding and string response parsing is UTF-8. You 
can override with charset of your choice:

```scala
DefaultHttp(url).charset("ISO-8859-1").asString
```

### Create your own HttpRequest builder

You don't have to use DefaultHttp. Create your own to set your own defaults:

```scala
object MyHttp extends BaseHttp (
  proxy: Proxy = Proxy.NO_PROXY, 
  options: Seq[HttpOptions.HttpOption] = HttpConstants.defaultOptions,
  charset: String = HttpConstants.utf8,
  sendBufferSize: Int = 4096,
  userAgent: String = "scalaj-http/1.0"
)
```