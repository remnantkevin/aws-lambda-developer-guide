# AWS Lambda function handler in C\#<a name="csharp-handler"></a>

The Lambda function *handler* is the method in your function code that processes events\. When your function is invoked, Lambda runs the handler method\. When the handler exits or returns a response, it becomes available to handle another event\.

You define a Lambda function handler as an instance or static method in a class\. If you want access to the Lambda context object, it is available by defining a method parameter of type *ILambdaContext*, an interface you can use to access information about the current invocation, such as the name of the current function, the memory limit, execution time remaining, and logging\. 

```
returnType handler-name(inputType input, ILambdaContext context) {
   ...
}
```

In the syntax, note the following:
+ *inputType* – The first handler parameter is the input to the handler, which can be event data \(published by an event source\) or custom input that you provide such as a string or any custom data object\. 
+ *returnType* – If you plan to invoke the Lambda function synchronously \(using the `RequestResponse` invocation type\), you can return the output of your function using any of the supported data types\. For example, if you use a Lambda function as a mobile application backend, you are invoking it synchronously\. Your output data type will be serialized into JSON\. 

  If you plan to invoke the Lambda function asynchronously \(using the `Event` invocation type\), the `returnType` should be `void`\. For example, if you use AWS Lambda with event sources such as Amazon S3 or Amazon SNS, these event sources invoke the Lambda function using the `Event` invocation type\.
+ `ILambdaContext context` – The second argument in the handler signature is optional\. It provides access to the [context object](csharp-context.md) which has information about the function and request\.

## Handling streams<a name="csharp-handler-streams"></a>

Only the `System.IO.Stream` type is supported as an input parameter by default\. 

For example, consider the following C\# example code\. 

```
using System.IO;

namespace Example
{            
  public class Hello
  {
    public Stream MyHandler(Stream stream)
    {
       //function logic
    }
  }
}
```

In the example C\# code, the first handler parameter is the input to the handler \(MyHandler\), which can be event data \(published by an event source such as Amazon S3\) or custom input you provide such as a `Stream` \(as in this example\) or any custom data object\. The output is of type `Stream`\. 

## Handling standard data types<a name="csharp-handler-types"></a>

All other types, as listed below, require you to specify a serializer\.
+ Primitive \.NET types \(such as string or int\)\.
+ Collections and maps \- IList, IEnumerable, IList<T>, Array, IDictionary, IDictionary<TKey, TValue>
+ POCO types \(Plain old CLR objects\)
+ Predefined AWS event types
+ For asynchronous invocations the return\-type will be ignored by Lambda\. The return type may be set to void in such cases\.
+ If you are using \.NET asynchronous programming, the return type can be Task and Task<T> types and use `async` and `await` keywords\. For more information, see [Using async in C\# functions with AWS Lambda](#csharp-handler-async)\.

Unless your function input and output parameters are of type `System.IO.Stream`, you will need to serialize them\. AWS Lambda provides a default serializers that can be applied at the assembly or method level of your application, or you can define your own by implementing the `ILambdaSerializer` interface provided by the `Amazon.Lambda.Core` library\. For more information, see [Deploy C\# Lambda functions with \.zip file archives](csharp-package.md)\.

Instruction how to define serialize you can find in [Serializing Lambda functions](#csharp-handler-serializer)\.

## Handler signatures<a name="csharp-handler-signatures"></a>

When creating Lambda functions, you have to provide a handler string that tells AWS Lambda where to look for the code to invoke\. In C\#, the format is:

 *ASSEMBLY::TYPE::METHOD* where:
+ *ASSEMBLY* is the name of the \.NET assembly file for your application\. When using the \.NET Core CLI to build your application, if you haven't set the assembly name using the `AssemblyName` property in \.csproj, the *ASSEMBLY* name will be the \.csproj file name\. For more information, see [\.NET Core CLI](csharp-package-cli.md)\. In this case, let's assume the \.csproj file is `HelloWorldApp.csproj`\.
+ *TYPE* is the full name of the handler type, which consists of the *Namespace* and the *ClassName*\. In this case `Example.Hello`\.
+ *METHOD* is name of the function handler, in this case `MyHandler`\.

Ultimately, the signature will be of this format: *Assembly::Namespace\.ClassName::MethodName*

Again, consider the following example:

```
using System.IO;

namespace Example
{            
  public class Hello
  {
    public Stream MyHandler(Stream stream)
    {
       //function logic
    }
  }
}
```

The handler string would be: `HelloWorldApp::Example.Hello::MyHandler` 

**Important**  
If the method specified in your handler string is overloaded, you must provide the exact signature of the method Lambda should invoke\. AWS Lambda will reject an otherwise valid signature if the resolution would require selecting among multiple \(overloaded\) signatures\. 

## Serializing Lambda functions<a name="csharp-handler-serializer"></a>

For any Lambda functions that use input or output types other than a `Stream` object, you will need to add a serialization library to your application\. You can do this in the following ways:
+ Use the `Amazon.Lambda.Serialization.SystemTextJson` NuGet package\. This library uses native .net core JSON serializer to handle serialization\. This package provides a performance improvement over `Amazon.Lambda.Serialization.Json`, but please note that there are some limitation documented in the [Microsoft's documentation](https://docs.microsoft.com/en-us/dotnet/standard/serialization/system-text-json-how-to?pivots=dotnet-core-3-1)\. It is only available for \.net core 3\.1 runtimes or higher\.
+ Use the `Amazon.Lambda.Serialization.Json` NuGet package\. This library uses `Newtonsoft.Json` NuGet package to handle serialization\.
+ Create your own serialization library by implementing the `ILambdaSerializer` interface, which is available as part of the `Amazon.Lambda.Core` library\. The interface defines two methods:
  + `T Deserialize<T>(Stream requestStream);`

     You implement this method to deserialize the request payload from the `Invoke` API into the object that is passed to the Lambda function handler\.
  + `T Serialize<T>(T response, Stream responseStream);`\.

     You implement this method to serialize the result returned from the Lambda function handler into the response payload that is returned by the `Invoke` API\.


To use serializer you need to add a dependency to your `csproj` file\.

```
 <ItemGroup>
    <PackageReference Include="Amazon.Lambda.Serialization.SystemTextJson" Version="2.1.0" />
    <!-- or -->
    <PackageReference Include="Amazon.Lambda.Serialization.Json" Version="2.0.0" />
  </ItemGroup>
```

Next you need to define the serializer. For example you can define it in the AssemblyInfo\.cs file\. If you are using `Amazon.Lambda.Serialization.SystemTextJson`, the serializercan be defined that way:

```
[assembly: LambdaSerializer(typeof(DefaultLambdaJsonSerializer))]
```
   
Using `Amazon.Lambda.Serialization.Json`, it will be:

```
[assembly: LambdaSerializer(typeof(JsonSerializer))]
```

You can also define a custom serialization attribute at the method level, which will override the default serializer specified at the assembly level\.

```
public class ProductService{

    [LambdaSerializer(typeof(JsonSerializer))]
    public Product DescribeProduct(DescribeProductRequest request)
    {
      return catalogService.DescribeProduct(request.Id);
    }
   
   [LambdaSerializer(typeof(MyJsonSerializer))]
   public Customer DescribeCustomer(DescribeCustomerRequest request)
   {
      return customerService.DescribeCustomer(request.Id);
   }
}
```

## Lambda function handler restrictions<a name="csharp-handler-restrictions"></a>

Note that there are some restrictions on the handler signature\.
+ It may not be `unsafe` and use pointer types in the handler signature, though `unsafe` context can be used inside the handler method and its dependencies\. For more information, see [unsafe \(C\# reference\)](https://msdn.microsoft.com/en-us/library/chfa2zb8.aspx)\.
+ It may not pass a variable number of parameters using the `params` keyword, or use `ArgIterator` as an input or return parameter which is used to support variable number of parameters\.
+ The handler may not be a generic method \(e\.g\. IList<T> Sort<T>\(IList<T> input\)\)\.
+ Async handlers with signature `async void` are not supported\.

## Using async in C\# functions with AWS Lambda<a name="csharp-handler-async"></a>

If you know your Lambda function will require a long\-running process, such as uploading large files to Amazon S3 or reading a large stream of records from DynamoDB, you can take advantage of the async/await pattern\. When you use this signature, Lambda invokes the function synchronously and waits for the function to return a response or for execution to time out\.

```
public async Task<Response> ProcessS3ImageResizeAsync(SimpleS3Event input)
{
   var response = await client.DoAsyncWork(input);
   return response;
}
```

If you use this pattern, there are some considerations you must take into account:
+ AWS Lambda does not support `async void` methods\.
+ If you create an async Lambda function without implementing the `await` operator, \.NET will issue a compiler warning and you will observe unexpected behavior\. For example, some async actions will run while others won't\. Or some async actions won't complete before the function invocation completes\.

  ```
  public async Task ProcessS3ImageResizeAsync(SimpleS3Event event) // Compiler warning
  {
      client.DoAsyncWork(input); 
  }
  ```
+ Your Lambda function can include multiple async calls, which can be invoked in parallel\. You can use the `Task.WhenAll` and `Task.WhenAny` methods to work with multiple tasks\. To use the `Task.WhenAll` method, you pass a list of the operations as an array to the method\. Note that in the example below, if you neglect to include any operation to the array, that call may return before its operation completes\.

  ```
  public async Task DoesNotWaitForAllTasks1()
  {
     // In Lambda, Console.WriteLine goes to CloudWatch Logs.
     var task1 = Task.Run(() => Console.WriteLine("Test1"));
     var task2 = Task.Run(() => Console.WriteLine("Test2"));
     var task3 = Task.Run(() => Console.WriteLine("Test3"));
   
     // Lambda may return before printing "Test2" since we never wait on task2.
     await Task.WhenAll(task1, task3);
  }
  ```

  To use the `Task.WhenAny` method, you again pass a list of operations as an array to the method\. The call returns as soon as the first operation completes, even if the others are still running\.

  ```
  public async Task DoesNotWaitForAllTasks2()
  {
    // In Lambda, Console.WriteLine goes to CloudWatch Logs.
    var task1 = Task.Run(() => Console.WriteLine("Test1"));
    var task2 = Task.Run(() => Console.WriteLine("Test2"));
    var task3 = Task.Run(() => Console.WriteLine("Test3"));
   
    // Lambda may return before printing all tests since we're only waiting for one to finish.
    await Task.WhenAny(task1, task2, task3);
  }
  ```
