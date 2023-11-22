# Javalin Simple Web Framework
(Summary at version 5.6.3)

* [REF](https://javalin.io/documentation)

* PRE-SETUP:
  ```
  dependency: io.javalin:javalin:5.6.3          or
              io.javalin:javalin-bundle:5.6.3  (including Jackson and Logback)
  ```


  ```kotlin
import io.javalin.Javalin

fun main() {

  val app = Javalin.create(config -> {  // <·· Instance config is divided in subgroups 
                                        //  
    // [[{security.AAA]]
    enum class Role : RouteRole { ANYONE, ROLE_ONE, ... }
    
    fun getUserRole(ctx: Context) : Array<Role> {
        // determine user role based on request.
        // typically done by inspecting headers, cookies, or user session
    }
                                           // AccessManager Subgroup
    config.accessManager {                 // <·· Example <<AccessManager>> implementation
      handler, ctx, routeRoles ->          //     allows to set per-endpoint AA.
                                           //     `before` filter allows for less restricted AA.
        val userRole = getUserRole(ctx)    //     <·· determine user role based on request
        if (routeRoles.contains(userRole)) {
            handler.handle(ctx)
        } else {
            ctx.status(401).result("Unauthorized")
        }
    }
    // [[security.AAA}]]

                                                         // Compression Config Subgroup
    config.compression.custom(compressionStrategy)       // set a custom CompressionStrategy
    config.compression.brotliAndGzip(gzipLvl, brotliLvl) // use both gzip and brotli (optional lvls)
    config.compression.gzipOnly(gzipLvl)                 // use gzip only (optional lvl)
    config.compression.brotliOnly(brotliLvl)             // use brotli only (optional lvl)
    config.compression.none()                            // disable compression

    // config subgroups:
    // config.http             // etags, request size, timeout, etc
    // config.routing          // context path, slash treatment
    // config.jetty            // jetty settings
    // config.staticFiles      // static files and webjars
    // config.spaRoot          // single page application roots
    // config.compression      // gzip, brotli, disable compression
    // config.requestLogger    // http and websocket loggers
    // config.plugins          // enable bundled plugins or add custom ones
    // config.vue              // vue settings, see /plugins/vue
    // config.contextResolvers // change implementation for Context functions
    // config.accessManager()  // configure the access manager
    // config.jsonMapper()     // configure the json mapper
    // config.fileRenderer()   // configure the file renderer
  })

  // QA: Error Control {{{
  app.exception(                         // <·· Exception Mapping
    NullPointerException::class.java,
    "html" /* Optional: html or JSON or */) {
      e, ctx -> ...
  }
  app.exception(Exception::class.java) { // <·· Will not trigger if more specific class is found
      e, ctx -> ...
  }
  
  app.error(404, "html")                 // <·· Error Code Mapping. Discouraged. No compiler support
  { ctx -> ctx.html(...)... }            //     "html" content type is optional.
  // QA: Error Control }}}

  app.attribute("myValue", "foo") // <·· Global App (Javalin instance) Attribute. 
                                  //     Read in Handlers like: ctx.appAttribute("myValue")
  app
    .before("/"){ ...log  code }             // <·· "befores" executed in order.
    .before("/"){ ...auth code }             //     matched before every request (even static files)
    .before("/"){ ...cache..   }             //     Known also as "filters", "interceptors" in other libraries.
    .before("/"){ 
       (ctx as JavalinServletContext)
      .tasks.clear() // <·· Access JavalinServletContext task queue, then skip
                     //     any remaining tasks for request (access-manager,
                     //     http-handlers, after-handlers, etc):
      }
    .after ("/"){ ...log  code }             // <·· run after every request (even on exceptions)
    .get("/path/*") { 
      ctx -> ctx.result(ctx.path() + " matches " + ctx.matchedPath())
    }
    .get("/un-secured",   { ctx -> ... },   Role.ANYONE  )
    .get("/secured",      { ctx -> ... },   Role.ROLE_ONE)
    .get("/")   { ctx -> ctx.result("Hi")  }
    //            { ctx -> ctx.status(201)   } // Alt 1
    //            { ctx -> ctx.json  (obj)   } // Alt 2
    //            { ctx -> ctx.future(future)} // Alt 3
    //            └──────────────────────────┴─ Handler implementation           
    //     └─┴─······························· Path. or /hello/{name}
    //                                         ┌─·············─┴────┘
    //                                         ctx.pathParam("name"))
    // └─┴─··································· Verb: get, post, put, patch, delete, head, options,
    //                                               trace, connect supported via Javalin#addHandler
    // 
  // Handler groups
  app.routes {          // <·· create (thread safe) temporal static instance of Javalin
                        //      allowing to skip app. prefix
      path("users") {   // <·· auto prefix with "/" (users == /users)
          get (...)
          post(...)
          path("{id}") {
              get(...)
              patch(...)
              delete(...)
          }
          ws("events", ...)
      }
      crud(            //  <·· crud "matches" (implements) 
       "a/{id}", ... ) //      getAll(ctx), getOne(ctx, resourceId), 
                       //      create(ctx) update(ctx, resourceId),
                       //      delete(ctx, resourceId)
  }
  app.start("0.0.0.0", 7070)      // <·· app..stop() stop server (sync/blocking)
  Runtime.getRuntime().addShutdownHook(new Thread(() -> {
  	app.stop();
  }));
  
  app.events(event -> {
    event.serverStopping(() -> { /* Your code here */ });
    event.serverStopped(() -> { /* Your code here */ });
    event.serverStarting { ... }
    event.serverStarted { ... }
    event.serverStartFailed { ... }
    event.handlerAdded { handlerMetaInfo -> }
    event.wsHandlerAdded { wsHandlerMetaInfo -> }
  });

  
}
```

## `Context`

* "provides everything needed to handle an http-request"
   It contains the underlying servlet-request and servlet-response.
* Request methods
  ```
  body()                                // request body as string
  bodyAsBytes()                         // request body as array of bytes
  bodyAsClass(clazz)                    // request body as specified class (deserialized from JSON)
  bodyStreamAsClass(clazz)              // request body as specified class (memory optimized version of above)
  bodyValidator(clazz)                  // request body as validator typed as specified class
  bodyInputStream()                     // the underyling input stream of the request
  uploadedFile("name")                  // uploaded file by name
  uploadedFiles("name")                 // all uploaded files by name
  uploadedFiles()                       // all uploaded files as list
  uploadedFileMap()                     // all uploaded files as a "names by files" map
  formParam("name")                     // form parameter by name, as string
  formParamAsClass("name", clazz)       // form parameter by name, as validator typed as specified class
  formParams("name")                    // list of form parameters by name
  formParamMap()                        // map of all form parameters
  pathParam("name")                     // path parameter by name as string
  pathParamAsClass("name", clazz)       // path parameter as validator typed as specified class
  pathParamMap()                        // map of all path parameters
  basicAuthCredentials()                // basic auth credentials (or null if not set)
  attribute("name", value)              // set an attribute on the request
  attribute("name")                     // get an attribute on the request
  attributeOrCompute("name", ctx -> {}) // get an attribute or compute it based on the context if absent
  attributeMap()                        // map of all attributes on the request
  contentLength()                       // content length of the request body
  contentType()                         // request content type
  cookie("name")                        // request cookie by name
  cookieMap()                           // map of all request cookies
  header("name")                        // request header by name (can be used with Header.HEADERNAME)
  headerAsClass("name", clazz)          // request header by name, as validator typed as specified class
  headerMap()                           // map of all request headers
  host()                                // host as string
  ip()                                  // ip as string
  isMultipart()                         // true if the request is multipart
  isMultipartFormData()                 // true if the request is multipart/formdata
  method()                              // request methods (GET, POST, etc)
  path()                                // request path
  port()                                // request port
  protocol()                            // request protocol
  queryParam("name")                    // query param by name as string
  queryParamAsClass("name", clazz)      // query param parameter by name, as validator typed as specified class
  queryParams("name")                   // list of query parameters by name
  queryParamMap()                       // map of all query parameters
  queryString()                         // full query string
  scheme()                              // request scheme
  sessionAttribute("name", value)       // set a session attribute
  sessionAttribute("name")              // get a session attribute
  consumeSessionAttribute("name")       // get a session attribute, and set value to null
  cachedSessionAttribute("name", value) // set a session attribute, and cache the value as a request attribute
  cachedSessionAttribute("name")        // get a session attribute, and cache the value as a request attribute
  cachedSessionAttributeOrCompute(...)  // same as above, but compute and set if value is absent
  sessionAttributeMap()                 // map of all session attributes
  url()                                 // request url
  fullUrl()                             // request url + query string
  contextPath()                         // request context path
  userAgent()                           // request user agent
  req()                                 // get the underlying HttpServletRequest
  ```

* Response methods
  ```
  result("result")                      // set result stream to specified string (overwrites any previously set result)
  result(byteArray)                     // set result stream to specified byte array (overwrites any previously set result)
  result(inputStream)                   // set result stream to specified input stream (overwrites any previously set result)
  future(futureSupplier)                // set the result to be a future, see async section (overwrites any previously set result)
  writeSeekableStream(inputStream)      // write content immediately as seekable stream (useful for audio and video)
  result()                              // get current result stream as string (if possible), and reset result stream
  resultInputStream()                   // get current result stream
  contentType("type")                   // set the response content type
  header("name", "value")               // set response header by name (can be used with Header.HEADERNAME)
  redirect("/path", code)               // redirect to the given path with the given status code
  status(code)                          // set the response status code
  status()                              // get the response status code
  cookie("name", "value", maxAge)       // set response cookie by name, with value and max-age (optional).
  cookie(cookie)                        // set cookie using javalin Cookie class
  removeCookie("name", "/path")         // removes cookie by name and path (optional)
  json(obj)                             // calls result(jsonString), and also sets content type to json
  jsonStream(obj)                       // calls result(jsonStream), and also sets content type to json
  html("html")                          // calls result(string), and also sets content type to html
  render("/template.tmpl", model)       // calls html(renderedTemplate)
  res()                                 // get the underlying HttpServletResponse
  ```
 
* Other methods
  ```
  async(runnable)                       // lifts request out of Jetty's ThreadPool, and moves it to Javalin's AsyncThreadPool
  handlerType()                         // handler type of the current handler (BEFORE, AFTER, GET, etc)
  appAttribute("name")                  // get an attribute on the Javalin instance. see app attributes section below
  matchedPath()                         // get the path that was used to match this request (ex, "/hello/{name}")
  endpointHandlerPath()                 // get the path of the endpoint handler that was used to match this request
  cookieStore()                         // see cookie store section below
  ```

## Cookie Store

* Cookies allow to share between handlers/requests/load-balanced servers.
```
ctx.cookieStore().set(key, value); // store any type of value
ctx.cookieStore().get(key);        // read any type of value
ctx.cookieStore().clear();         // clear the cookie-store
```
### Rules:

* max-size: 4kB
* The first handler that matches the incoming request will populate 
  the cookie-store-map with the data currently stored in the cookie (if 
  any).
* This map can now be used as a state between handlers on the same 
  request-cycle, pretty much in the same way as ctx.attribute()
* At the end of the request-cycle, the cookie-store-map is 
  serialized, base64-encoded and written to response as a cookie. 
  allowing to share the map between requests and servers (in case
  you are running multiple servers behind a load-balancer).

## WebSockets

```
app.ws("/websocket/{path}") { ws ->
    ws.onConnect { ctx -> println("Connected") }
    ws.onConnect      (WsConnectContext      )
    ws.onError        (WsErrorContext        )
    ws.onClose        (WsCloseContext        )
    ws.onMessage      (WsMessageContext      )
    ws.onBinaryMessage(WsBinaryMessageContext)
 //                    └────────────────────┘
 // different WsContext's flavors expose different things.
    
app.wsBefore         { ws -> ... } // runs before all WebSocket requests
app.wsBefore("/a/*") { ws -> ... } // runs before websocket requests to /path/*
//                    └──┴─ WsContext
app.wsAfter ....
}
```

## Validate query, form, path params headers and request body: [[{QA.error_mng]]

```
                                             // create Validator<ClassA> for:
  ctx.queryParamAsClass<ClassA>("paramName") // the value of queryParam("paramName")
  ctx.formParamAsClass<ClassA>("paramName" ) // the value of formParam("paramName")
  ctx.pathParamAsClass<ClassA>("paramName" ) // the value of pathParam("paramName")
  ctx.headerAsClass   <ClassA>("headerName") // the value of header("paramName")
  ctx.bodyValidator   <ClassA>()             // the value of body()
  
  Validator.create(clazz, value, fieldName). // <·· Custom validators
  Validator API:
  allowNullable()                   // turn into NullableValidator (must be called first)
  check(predicate, "error")         // add check with a ValidationError("error") to Validator
  check(predicate, validationError) // add check with a ValidationError to Validator, (opt) args for l10n
  get()                             // return validated value as specified type, or throw ValidationException
  getOrThrow(exceptionFunction)     // return the validated value as the specified type, or throw custom exception
  getOrDefault()                    // return default-value if value is null, else call get()
  errors()                          // get all errors of Validator (as map("fieldName", List<ValidationError>))
  
  // VALIDATE A SINGLE QUERY PARAMETER WITH A DEFAULT VALUE /////////////////////////////////////////////
  val myValue = ctx.queryParamAsClass<Int>("value")   // validate value
                   .getOrDefault(788)                 // default value.
  ctx.result(value) // return validated value to the client
  // GET ?value=a would yield HTTP 400 - {"my-qp":[{"message":"TYPE_CONVERSION_FAILED","args":{},"value":"a"}]}
  // GET ?value=1 would yield HTTP 200 - 1 (the validated value)
  // GET ?        would yield HTTP 200 - 788 (the default value)

// VALIDATE TWO DEPENDENT QUERY PARAMETERS ////////////////////////////////////////////////////////////
val fromDate = ctx.queryParamAsClass<Instant>("from").get()
val toDate = ctx.queryParamAsClass<Instant>("to")
    .check({ it.isAfter(fromDate) }, "'to' has to be after 'from'")
    .get()

// VALIDATE A JSON BODY ///////////////////////////////////////////////////////////////////////////////
val myObject = ctx.bodyValidator<MyObject>()
    .check({ it.myObjectProperty == someValue }, "THINGS_MUST_BE_EQUAL")
    .get()

// VALIDATE WITH CUSTOM VALIDATIONERROR ///////////////////////////////////////////////////////////////
ctx.queryParamAsClass<Int>("param")
    .check({ it > 5 }, ValidationError("OVER_LIMIT", args = mapOf("limit" to 5)))
    .get()
// GET ?param=10 would yield HTTP 400 - {"param":[{"message":"OVER_LIMIT","args":{"limit":5},"value":10}]}


Collecting multiple errors

val ageValidator = ctx.queryParamAsClass<Int>("age")
    .check({ !it.contains("-") }, "ILLEGAL_CHARACTER")

// Empty map if no errors, otherwise a map with the key "age" and failed check messages in the list.
val errors = ageValidator.errors()

// Merges all errors from all validators in the list. Empty map if no errors exist.
val manyErrors = listOf(ageValidator, otherValidator, ...).collectErrors()

ValidationException

When a Validator throws, it is mapped by:

app.exception(ValidationException::class.java) { e, ctx ->
    ctx.json(e.errors).status(400)
}

You can override this by doing:

app.exception(ValidationException::class.java) { e, ctx ->
    // your code
}


JavalinValidation             // <·· to validate a non-included class, register custom converter:
  .register(Instant::class.java)
    { Instant.ofEpochMilli(it.toLong()) }
```
[[}]]

## Default responses

app.post("/forbidden") {
  throw new ForbiddenResponse("Off limits!")  <·· Map<String, String> can be used instead.
  // Returns:
  // for clients           for clients
  // not accepting JSON    accepting JSON
  //                        
  // Forbidden             {
  //                           "title": "Off limits!",
  //                           "status": 403,
  //                           "type": "https://javalin.io/documentation#forbiddenresponse",
  //                           "details": []
  //                       }
  // 
  //                                   Response returned
  // throw RedirectResponse            302 Found 
  // throw BadRequestResponse          400 Bad Request 
  // throw UnauthorizedResponse        401 Unauthorized 
  // throw ForbiddenResponse           403 Forbidden 
  // throw NotFoundResponse            404 Not Found
  // throw MethodNotAllowedResponse    405 Method Not Allowed
  // throw ConflictResponse            409 Conflict response 
  // throw GoneResponse                410 Gone 
  // throw InternalServerErrorResponse 500 Internal Server Error 
  // throw BadGatewayResponse          502 Bad Gateway
  // throw ServiceUnavailableResponse  503 Service Unavailable 
  // throw GatewayTimeoutResponse      504 Gateway Timeout 
  // 
}

## Server-sent ("Event Source") Events

```
val clients = ConcurrentLinkedQueue<SseClient>()
app.sse("/sse") {  // app.sse() provides access to the connected SseClient
  client ->
    client.sendEvent("connected", "Hello, SSE")
    // Alt1: Clients automatically closed when leaving the handler
    client.onClose { println("Client disconnected") }
    client.close() 
    // Alt2: use client.keepAlive() to reuse the client outside the handler
    client.keepAlive()
    client.onClose { clients.remove(client) }
    clients.add(client)
}
```

## Tunning Config  [[{]]
    config.http.generateEtags = booleanValue     // should javalin generate etags for dyn.responses
    config.http.prefer405over404 = booleanValue  
    config.http.maxRequestSize = longValue       
    config.http.defaultContentType = stringValue
    config.http.asyncTimeout = longValue         // 0=> No timeout 


    config.requestLogger.http { ctx, ms /*time to finish request */ -> ...  }

    config.staticFiles.add { staticFiles ->
        staticFiles.hostedPath = "/"             
        staticFiles.directory = "/public"        
        staticFiles.location = Location.CLASSPATH
        staticFiles.precompress = false          
        staticFiles.aliasCheck = null            
        staticFiles.headers = mapOf(...)         
        staticFiles.skipFileFunction = { req -> false }
        staticFiles.mimeTypes.add(mimeType, ext)
      }

    config.jetty.server(serverSupplier)                 // set Jetty Server for Javalin to run on
    config.jetty.sessionHandler(sessionHandlerSupplier) 
    config.jetty.contextHandlerConfig(contextHandlerConsumer) // configure ServletContextHandler 
    config.jetty.wsFactoryConfig(jettyWebSocketServletFactoryConsumer) 
    config.jetty.httpConfigurationConfig(httpConfigurationConsumer)

    config.jetty.multipartConfig.cacheDirectory("/tmp")
    config.jetty.multipartConfig.maxFileSize(100, SizeUnit.MB)
    config.jetty.multipartConfig.maxInMemoryFileSize(10, SizeUnit.MB)
    config.jetty.multipartConfig.maxTotalRequestSize(1, SizeUnit.GB)
  
    config.routing.contextPath           = stringValue  // ex '/blog' to host all app in subpath.
    config.routing.ignoreTrailingSlashes = booleanValue // true => '/path' == '/path/'
    config.routing.treatMultipleSlashesAsSingleSlash = booleanValue // true => '/path//' == '/path/'
    config.routing.caseInsensitiveRoutes = booleanValue 

    config.contextResolver.ip      = { ctx -> "custom ip" }  
    config.contextResolver.host    = { ctx -> "custom host" }
    config.contextResolver.scheme  = { ctx -> "custom scheme" }
    config.contextResolver.url     = { ctx -> "custom url" }
    config.contextResolver.fullUrl = { ctx -> "custom fullUrl" }
[[}]]


## Logging

* No logger included by default. Simple "fix":
  <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-simple</artifactId>
      <version>2.0.7</version>
  </dependency>

## Others
* See original doc for SSL/HTTP2 into Javalin, Plugins, Request lifecycle,
  Rate limiting, Android, WebSocket Message Ordering, Uploaded files , 
  Asynchronous requests, Adding other Servlets and Filters to Javalin,
  Views and Templates, simplified Vue.js development support, Custom classloader

## Concurrency [[{]]
* Javalin uses a Jetty QueuedThreadPool with 250 threads for serving requests by default.
  (=> up to 250 concurrent requests)
* Each incoming request is handled by a dedicated thread, so all Handler implementations
  should be thread-safe.
* Use Asynchronous requests for long-running requests or setup with Java 21 virtual threads.
* Default non-async configuration provides similar performance to Jetty, which can handle over a
  million plaintext requests per second.

## Tomcat (and others) integration:
* Javalin is primarily meant to be used with the embedded Jetty          [[{PM.TODO]]
  server, but if you want to run Javalin on another web server (such as 
  Tomcat), you can use Maven or Gradle to exclude Jetty, and attach 
  Javalin as a servlet.                                                  [[PM.TODO}]]
[[}]]

## JSON mapper  [[{]]
* you need to pass an object which implements <<JsonMapper>> to config.jsonMapper().


```
   <<JsonMapper>>
   String      toJsonString(Object obj, Type type)                    // basic method for mapping to json
   InputStream toJsonStream(Object obj, Type type)                    // more memory efficient method for mapping to json
   writeToOutputStream(Stream<*> stream, OutputStream outputStream)   // most memory efficient method for mapping to json
   // 
   <T> T fromJsonString(String json, Type targetType)                 // basic method for mapping from json
   <T> T fromJsonStream(InputStream json, Type targetType)            // more memory efficient method for mapping from json
```
* Defaults to Jackson - fast and feature-rich.

  ```
  config.jsonMapper(
    JavalinJackson().updateMapper { mapper ->
      mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL)
  })

  // GSON example alternative

  val gson = GsonBuilder().create()
  val gsonMapper = object : JsonMapper {
      override fun <T : Any> fromJsonString(json: String, targetType: Type): T =
          gson.fromJson(json, targetType)
      override fun toJsonString(obj: Any, type: Type) =
          gson.toJson(obj)
  }
  val app = Javalin.create { it.jsonMapper(gsonMapper) }.start(7070)
  ```
[[}]]


## Java lang Error handling

* Default error handler for java.lang.Error will log the error and return a 500.
  Override it like:
  ```
  Javalin.create { cfg ->
      cfg.pvt.javaLangErrorHandler { res, error ->
          res.status = HttpStatus.INTERNAL_SERVER_ERROR.code
          JavalinLogger.error("Exception occurred while servicing http-request", error)
      }
  }
  ```

# Relevant issues
* https://github.com/javalin/javalin/issues/358 (with solution)
* https://github.com/javalin/javalin/issues/232
* https://github.com/javalin/javalin/issues/1462

