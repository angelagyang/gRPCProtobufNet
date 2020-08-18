# Reusing WCF Data Contracts for gRPC (protobuf-net v2.4.4)
When using protobuf-net, we need to modify our data contracts (e.g. adding the proper attributes) to ensure correct serialization. This page provides some key considerations to keep in mind.


## Ordering
DataMembers must be explicitly ordered. The [integer ordering serves as identification](https://github.com/protobuf-net/protobuf-net#1-first-decorate-your-classes) for each individual member in serialization. 

**Important Note**: For backward-compatibility with WCF services, refer to [these guidelines].(https://docs.microsoft.com/en-us/dotnet/framework/wcf/feature-details/data-member-order). 

**Existing contract for WCF services:** 
```csharp
[DataContract]
class Student {
    [DataMember]
    public int Age;
    [DataMember]
    public string Name;
}

[DataContract]
class Class {
    [DataMember]
    public int RoomNumber;
    [DataMember]
    public string Subject;
    [DataMember]
    public List<Student> Students;
}

```

**Updated contract for compatibility with protobuf-net and WCF:** 
```csharp
[DataContract]
class Student {
    [DataMember(Order = 1)]
    public int Age;
    [DataMember(Order = 2)]
    public string Name;
}

[DataContract]
class Class {
    [DataMember(Order = 1)]
    public int RoomNumber;
    [DataMember(Order = 3)]
    public string Subject;
    [DataMember(Order = 2)]
    public List<Student> Students;
}

```

## Inheritance
[Inheritance must be explicitly declared](https://github.com/protobuf-net/protobuf-net/blob/main/README.md#inheritance) with ProtoInclude. 
**Important Note**: The integer key used in ProtoInclude must be unique and not used in the DataMembers/ProtoMembers (i.e. in this case, 1 or 2 could not be used). 


**Existing contract for WCF services:** 
```csharp
[DataContract]
class Student {
    [DataMember]
    public int Age;
    [DataMember]
    public string Name;
}

[DataContract]
class CollegeStudent : Student {
    [DataMember]
    public string CollegeName;
}

```

**Updated contract for compatibility with protobuf-net and WCF:** 
```csharp
[DataContract]
[ProtoInclude(5, typeof(CollegeStudent)]
class Student {
    [DataMember(Order = 1)]
    public int Age;
    [DataMember(Order = 2)]
    public string Name;
}

[DataContract]
class CollegeStudent : Student {
    [DataMember(Order = 1)]
    public string CollegeName;
}

```

## Parameterless constructors 
Make sure your DataContract class has a parameterless constructor. By default, it should. However, if it is overriden by a non-parameterless constructor, you must explicitly declare the parameterless constructor. Without it, you may see: ```Grpc.Core.RpcException: "Failed to deserialize response message..."```

```csharp
[DataContract]
class Student {
    [DataMember(Order = 1)]
    public int Age;
    [DataMember(Order = 2)]
    public string Name;
    
    public Student(int age) {
      this.age = age;
    }
    public Student() {} // explicit declaration of parameterless constructor
}
```

## Methods with primitive parameters are NOT supported 
See [this Github issue](https://github.com/protobuf-net/protobuf-net.Grpc/issues/70) for more context. Primitive parameters will need to be wrapped in some object. 

**Sample WCF Service Contract:** 
```csharp
[ServiceContract]
interface IStudentService {
   [OperationContract]
   bool IsAdult(int studentId)
}
```

**Updated Service Contract for protobuf-net/gRPC compatibility:**
```csharp
[ServiceContract]
interface IStudentService {
   [OperationContract]
   BoolResponse IsAdult(int studentId)
}

[DataContract]
class BoolResponse {
   [DataMember(Order = 1)]
   public bool Value; 
}
```

# Adding gRPC middle-ware for exceptions and logging 
Exceptions thrown on the server-side will not be fully visible to the client. Instead, clients will throw a ```RpcException```. If you want to preserve the original exception, then there are some options. 

### EnableDetailedErrors = true 
Setting this config will append the original exception message to the ```RpcException``` thrown on the client-side. 
```csharp 
services.AddCodeFirstGrpc(config =>
{
    config.EnableDetailedErrors = true;
    config.ResponseCompressionLevel = System.IO.Compression.CompressionLevel.Optimal;
});
```

### Configure an interceptor for error-handling
You can also configure a gRPC interceptor to catch server-side exceptions, serialize the exception metadata, and pack it into a custom ```RpcException```.

Helpful references: 
* [StackOverflow response](https://stackoverflow.com/questions/62823340/how-to-get-grpc-to-throw-the-original-exception-like-wcf-does/62944504#62944504)
* [ASP Grpc Custom Error Handling](https://digitteck.com/dotnet/aspnetcore/asp-grpc-custom-error-handling/) 

**Configure the interceptor:**
```csharp 
services.AddGrpc(options => 
{
    options.Interceptors.Add<GRPCInterceptor>();     
});
```

**Bare-bones interceptor for logging:**
```csharp 
public class GRPCInterceptor : Interceptor
{
    private readonly ILogger logger;

    public GRPCInterceptor(ILogger logger)
    {
        this.logger = logger;
    }

    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(TRequest request, ServerCallContext context, UnaryServerMethod<TRequest, TResponse> continuation)
    {
        logger.Debug($"starting call");

        var response = await base.UnaryServerHandler(request, context, continuation);

        logger.Debug($"finished call");

        return response;
    }
}
```

**Interceptor for error-handling:**
```csharp
public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(TRequest request, ServerCallContext context, UnaryServerMethod<TRequest, TResponse> continuation)
{
    TResponse response; 
    try
    {
        response = await continuation(request, context);
    } 
     catch (Exception e) // Add your own exception handling logic 
     {
        if (e.Message != null)
        {
            throw ExtractRpcException(e.Message, e.Code); 
        }
        throw; 
     }
}

public static RpcException ExtractRpcException(String exMessage, String code)
{
    var metadata = new Metadata
    {
        { "Code", code },
        { "Message", message } 
     };
     return new RpcException(new Grpc.Core.Status(StatusCode.Internal, exMessage), metadata);
}
 ```
 
# References
* [protobuf-net repository](https://github.com/protobuf-net/protobuf-net) 
