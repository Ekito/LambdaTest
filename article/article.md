# What's the fuss about FaaS?

A recent article about [Serverless Architectures](http://martinfowler.com/articles/serverless.html#benefits) by Mike Roberts 
as well as ekito's commitment to developing [nowave.io](https://nowave.io), whose architecture is partly a serverless, 
inspired me on having my own take on Serverless Architectures based on FaaS (Functions as a Service).

At first I will describe how I developed a Hello World FaaS that is triggered by a REST API. 
The API is declaratively defined using a [Swagger](http://swagger.io/) description in a contract first approach. 
The API is deployed to Amazon's [API Gateway](https://aws.amazon.com/api-gateway/). In the gateway, a mapping is 
configured extracting data from API requests and transforming them to event objects that are passed to Amazon's FaaS 
infrastructure called ([Amazon Lambda](https://aws.amazon.com/lambda/details/)).

In the second part I will provide some feedback on load testing the service I developed. This part includes some 
insights to observed round trip latencies and rate limiting behaviours.

But first what's all that fuss about FaaS?

## FaaS
A Function as a Service (FaaS) is some stateless business logic that is triggered by a stimulus, for example an event. The 
logic of the function is applied to its inputs; the result is returned. FaaS can have side effects. For example, they 
can trigger another function or communicate with third party services. 

For FaaS to execute, they are bound to triggers. Therefore they become versatile building blocks that can act as 
a stored procedure on database tables, be triggered on file system changes, act as workers on messaging queues 
or act as controllers behind a REST API.

FaaS are packaged and deployed to a cloud provider. FaaS usage is generally billed by the number of function invocations 
and by computing resources consumed during the execution of a function call.

What makes FaaS so attractive is their simple programming, packaging and deployment model, but also their capacity of 
being natively elastic, resulting in linear scalability in terms of performance and operational costs. Furthermore, 
the FaaS deployment model does not require any execution platform, middleware, container or virtual machine. Operational 
maintenance is completely outsourced to the cloud provider (NoOps).

## Building the HelloWorld function

Amazon Lambda accepts functions implemented in JavaScript (Node.js), Python or Java 8. By consequence, it is also 
possible to write a function in Scala or any other programming language that is based on Java 8.

Willing to write a little bit of Scala again, I decided to follow this 
[blog entry](https://aws.amazon.com/blogs/compute/writing-aws-lambda-functions-in-scala/) for implementing my function.

### Project Initialization
First, let's initialize a Scala project with the following `build.sbt` file at the project's root:

```
name := "AWSLambdaTest"

version := "1.0"

scalaVersion := "2.11.8"

libraryDependencies += "com.amazonaws" % "aws-lambda-java-core" % "1.1.0"
libraryDependencies += "com.fasterxml.jackson.module" % "jackson-module-scala_2.11" % "2.7.5"

javacOptions ++= Seq("-source", "1.8", "-target", "1.8", "-Xlint")

assemblyMergeStrategy in assembly := {
  case PathList("META-INF", xs@_*) => MergeStrategy.discard
  case x => MergeStrategy.first
}
```

Note here that we instruct sbt expressively to build Java 8 byte code. Furthermore, the assembly strategy for the [SBT 
assembly plugin](https://github.com/sbt/sbt-assembly) is configured.

The assembly plugin is configured in `LambdaTest/project/plugins.sbt` as follows:

```
addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "0.14.3")
```

The project is built and packaged with the command `sbt assembly`. The resulting jar can be found at 
`LambdaTest/target/scala-2.11/AWSLambdaTest-assembly-1.0.jar`. 

What stroke me was the size of the jar file (12.4MB) given that we are writing a mere hello world function. It turns out 
that all Scala byte code classes are included in the jar. In practical terms this size can have an
impact on the upload duration to AWS (timeout after 61s) and the 
[number of functions](http://docs.aws.amazon.com/lambda/latest/dg/limits.html) that can be deployed per user account and 
region (max. storage space for all lambdas in one region is 1.5GB). Should this really become impacting, consider 
deploying to different regions, using directly Java 8 or an interpreted language with a smaller footprint (Node.js or 
Python).
 
### Programming model

Functions are implemented within a handler with a name of your choice. The function signature can take several forms, 
always defining an input and an output. Amazon Lambda allows for the following
[input and output](http://docs.aws.amazon.com/lambda/latest/dg/java-programming-model-handler-types.html) types:

- Simple Java types (AWS Lambda supports the String, Integer, Boolean, Map, and List types)
- POJO (Plain Old Java Object) type
- Stream types (InputStream / OutputStream)

When invoking a Lambda function asynchronously, any return value by the Lambda function will be ignored. It is 
encouraged to use the `void` return type.

When using POJOs, Amazon Lambda serializes and deserializes based on standard bean naming conventions. 
Mutable POJOs with public getters and setters must be used. Annotations are ignored.

When developing AWS Lambdas in Scala, type conversions between Scalaisms and Javaisms must be implemented. 
Alternatively, stream types for inputs and outputs can be used. In that case, marshalling and unmarshalling JSON is
realized within the Lambda. While this approach seems a little cumbersome it is in my opinion the most stable strategy 
when opting for Scala. I am not inventing anything here, its coming straight from the
[tutorial](https://aws.amazon.com/blogs/compute/writing-aws-lambda-functions-in-scala/) I was following.

```
package io.ekito.lambdaTest

case class NameInfo(firstName: String, lastName: String)

class Main {

  import java.io.{InputStream, OutputStream}


  val scalaMapper = {
    import com.fasterxml.jackson.databind.ObjectMapper
    import com.fasterxml.jackson.module.scala.DefaultScalaModule
    new ObjectMapper().registerModule(new DefaultScalaModule)
  }

  def hello(input: InputStream, output: OutputStream): Unit = {
    val name = scalaMapper.readValue(input, classOf[NameInfo])
    val result = s"Greetings ${name.firstName} ${name.lastName}."
    output.write(result.getBytes("UTF-8"))
  }
}
```

The complete source code is available on [Github](https://github.com/Ekito/LambdaTest).

### Deployment

With the code in place, call:
 ```
$> sbt assemby 
 ```
 
Now lets deploy the artifact. You can do this via the CLI or the console; below I’ve illustrated what it 
looks like in the AWS console. Skip the steps 'Select BluePrint' and 'Configure Triggers' to arrive at the 
following screen:

<screen 1>

I am using the default suggestions for memory, duration, and role (basic execution). Note the handler: 
`io.ekito.lambdaTest.Main::hello`. Click 'Create Lambda function' and you should be ready to test.

### Testing
At this point the function should be fully operational. First configure a JSON object of the test event:

<screen_test_event>

Now you can test it by sending a JSON event object as illustrated 
here:
<screen_test_result>

What's interesting here is information about execution duration, billed duration the used memory. Based on those values, 
we can reduce the amount of configured memory from 512MB to 128MB. Interesting is also that an initial invocation 
can take up to 2s. There is somewhat of a warmup taking place. Execution times upon subsequent invocations are pretty 
short: ~ 0.6ms. However, the smallest billed period being 100ms, Amazon bills us for resources that we are actually 
not using :-|

### REST API definition
Using Swagger's API description language, lets wrtie a little Hello World REST API with one GET operation. To do so, 
I used the [Swaggerhub.com](https://swaggerhub.com) online editor. SwaggerHub provides inline syntax and grammar 
validation as well as API documentation preview.
 
```
swagger: '2.0'
info:
  version: '1'
  title: Lambda Test
  description: A hello world API testing AWS API Gateway + Lambda

paths:
  /hello/{firstName}:
    get:
      consumes:
      - application/json
      produces:
      - text/html
      responses:
        200:
          description: "200 response"
          headers:
            Content-Type:
              type: string
      parameters:
        - name: firstName
          in: path
          description: first name
          required: true
          type: string
```

### API Gateway
In Amazon API Gateway, create a new API by pasting the previously created API definition.

<screen api definition>

Now, link the operation `/hello/{firstName}/GET`, with our lambda function.

<screen api linking>

With the API operation being linked to the lambda function a final step is necessary: The request's parameters must be
transformed into a JSON event object, compatible with the input parameter of our lambda's handler function. The call 
flow is visualized hereafter:

<screen call flow>

Click on 'Integration Request' inorder to define the mapping. Several mappings can be defined depending on the request's 
`Content-Type` header argument. We are defining one template for the content type `application/json`.

<screen mapping>

The mapping can include all request parameters, payloads or header attributes. In our example, we are injecting the 
path variable `firstName`. The full documentation for request and response mapping can be 
[found here](http://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-mapping-template-reference.html).

With the mapping in place, we can now test the integration:

<screen test api gateway>

A final step consists in deploying the API. Deployments are created in stages allowing for deployment of different 
versions of the same API. With the seamless scalability of the service, Rate and burt limits are crucial to define the
capacity of your service and to protect your bank account ;-). On the top of the page, the public URL is displayed.

<screen deployment>

## Load testing

Now its time to load test our public FaaS service. But wait, Amazon's API gateway uses SSL termination with TLSV1.2 
encryption and [Server Name Indication](https://en.wikipedia.org/wiki/Server_Name_Indication). SNI a TLS protocol extension that 
unfortunately not a lot of load testing tools support. As of the time of writing
[Apache Benchmarking Tool](https://httpd.apache.org/docs/2.4/programs/ab.html) and 
[goBench](https://github.com/cmpxchg16/gobench) did not work for me. I got more lucky with [wrk](https://github.com/wg/wrk).

##Test Setup
I configured the API endpoint with a rate limit of 100 and a burst of 200 requests / second. 

Different test runs have been done through a non-saturated home ADSL connection. Apart from my home ADSL connection,
I also spawned a Digital Ocean droplet in the London Data Center. The API Gateway was configured in the eu-west region 
in Ireland. Test results from both sites (Toulouse Area via local ISP, London Area Digital Ocean Droplet) are comparable.

##Test 1: 8 concurrent connections
80.5 req/s. We are getting close to the the rate limit. Latency for the 90th percentile appears to be high.

```
➜  virtualdisks docker run --rm williamyeh/wrk -c 8 -t 1 -d 60s --latency https://ph6oo2np9a.execute-api.eu-west-1.amazonaws.com/prd/hello/bert
Running 1m test @ https://ph6oo2np9a.execute-api.eu-west-1.amazonaws.com/prd/hello/bert
  1 threads and 8 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   124.92ms  144.94ms   1.41s    93.52%
    Req/Sec    82.07     23.87   141.00     70.74%
  Latency Distribution
     50%   77.97ms
     75%  106.22ms
     90%  160.59ms
     99%  878.88ms
  4836 requests in 1.00m, 1.78MB read
Requests/sec:     80.52
Transfer/sec:     30.27KB
```

## Test 2: 14 concurrent connections
As expected, the rate limit kicks in. We have 2993 of 9146 requests with a non-2xx or 3xx response. 
Latency at the 90th percentile is 144 ms.

`9146 - 2993 = 6153 req/minute = 102.5 req/s`

```➜  virtualdisks docker run --rm williamyeh/wrk -c 14 -t 1 -d 60s --latency https://ph6oo2np9a.execute-api.eu-west-1.amazonaws.com/prd/hello/bert
   Running 1m test @ https://ph6oo2np9a.execute-api.eu-west-1.amazonaws.com/prd/hello/bert
     1 threads and 14 connections
     Thread Stats   Avg      Stdev     Max   +/- Stdev
       Latency   118.14ms  146.95ms   1.48s    93.70%
       Req/Sec   153.46     34.74   240.00     72.36%
     Latency Distribution
        50%   75.28ms
        75%   87.52ms
        90%  144.13ms
        99%  892.30ms
     9146 requests in 1.00m, 3.38MB read
     Non-2xx or 3xx responses: 2993
   Requests/sec:    152.29
   Transfer/sec:     57.59KB
```

## Analysis
Different load tests showed that rate limiting in the API Gateway works as expected. Interestingly, the overall latency 
appears to be relatively high (generally > 130ms in the 90th percentile).

Tests have been conducted through two distinct ISPs with similar results. Given that our lambda operates in the 
sub-millisecond range, latency must come from the API Gateway's SSL termination, routing and mapping infrastructure.

# Conclusion
In conclusion, the API Gateway is a precious component in a Serverless Architecture, allowing the declarative 
construction of an API façade in front of BaaS and FaaS components. It also integrates well with third party 
authentication providers, such as (Auth0)[https://auth0.com/]. However, its latency surprised me a little bit.

Building a FaaS with Amazon Lambda is quite easy. Its an amazing building block for all kinds of architectures, 
including Serverless ones.

While this blog post only showed its deployment through the AWS console, deployment can also be automated through 
Amazon's command line interface and deployment APIs, allowing for eased integration into a Continuous Delivery pipeline.