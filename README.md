# Advanced Mapster Features and Mapping Scenarios

## Table of Contents
- [Basic Controller Implementation](#basic-controller-implementation)
- [Complex Mapping Scenarios](#complex-mapping-scenarios)
- [Custom Property Mapping](#custom-property-mapping)
- [Conditional Mapping](#conditional-mapping)
- [Navigation Property Mapping](#navigation-property-mapping)
- [Best Practices](#best-practices)

## Basic Controller Implementation

```csharp
[ApiController]
[Route("api/[controller]")]
public class PollsController : ControllerBase
{
    private readonly IPollService _pollService;
    private readonly IMapper _mapper;

    public PollsController(IPollService pollService, IMapper mapper)
    {
        _pollService = pollService;
        _mapper = mapper;
    }

    // GET: api/polls
    [HttpGet]
    public IActionResult GetAll()
    {
        var polls = _pollService.GetAll();
        var response = polls.Adapt<IEnumerable<PollResponse>>();
        return Ok(response);
    }
    // Response: [{"id":1,"title":"Poll 1","description":"Description 1"},...]

    // POST: api/polls
    [HttpPost]
    public IActionResult Add([FromBody] CreatePollRequest request)
    {
        var newPoll = _pollService.Add(request.Adapt<Poll>());
        return CreatedAtAction(
            nameof(Get), 
            new { id = newPoll.Id }, 
            newPoll.Adapt<PollResponse>());
    }
    // Response: {"id":1,"title":"New Poll","description":"New Description"}

    // PUT: api/polls/{id}
    [HttpPut("{id}")]
    public IActionResult Update(
        [FromRoute] int id, 
        [FromBody] CreatePollRequest request)
    {
        var isUpdated = _pollService.Update(id, request.Adapt<Poll>());
        return !isUpdated ? NotFound() : NoContent();
    }
    // Response: 204 No Content
}
```

## Complex Mapping Scenarios

### Domain Models
```csharp
public class Student
{
    public int Id { get; set; }
    public string FirstName { get; set; } = string.Empty;
    public string MiddleName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public DateTime? DateOfBirth { get; set; }
    public Department Department { get; set; } = default!;
}

public class Department
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
}
```

### DTOs
```csharp
public class StudentResponse
{
    public int Id { get; set; }
    public string FullName { get; set; } = string.Empty;
    public int? Age { get; set; }
    public string DepartmentName { get; set; } = string.Empty;
}
```

### Mapping Configuration
```csharp
public class MappingConfigurations : IRegister
{
    public void Register(TypeAdapterConfig config)
    {
        config.NewConfig<Student, StudentResponse>()
            // Combine multiple properties into one
            .Map(dest => dest.FullName, 
                src => $"{src.FirstName} {src.MiddleName} {src.LastName}")
            
            // Calculate age with null checking
            .Map(dest => dest.Age, 
                src => DateTime.Now.Year - src.DateOfBirth!.Value.Year,
                srcCond => srcCond.DateOfBirth.HasValue)
            
            // Map from nested object
            .Map(dest => dest.DepartmentName, src => src.Department.Name);

        // Enable two-way mapping
        config.NewConfig<Student, StudentResponse>()
            .TwoWays();
    }
}
```

### Test Endpoint
```csharp
[HttpGet("test")]
public IActionResult Test()
{
    var student = new Student
    {
        Id = 1,
        FirstName = "Mohammad",
        MiddleName = "Abd-El-Shafy",
        LastName = "KhairAllah",
        DateOfBirth = new DateTime(1994, 12, 28),
        Department = new Department
        {
            Id = 1,
            Name = "Test"
        }
    };

    var response = student.Adapt<StudentResponse>();
    return Ok(response);
}
// Response: {
//   "id": 1,
//   "fullName": "Mohammad Abd-El-Shafy KhairAllah",
//   "age": 29,
//   "departmentName": "Test"
// }
```

## Best Practices

1. **Property Name Conventions**
   ```csharp
   // Mapster will automatically map DepartmentName to Department.Name
   // No need for explicit configuration unless different naming
   public string DepartmentName { get; set; }
   ```

2. **Null Handling**
   ```csharp
   .Map(dest => dest.Age, 
       src => DateTime.Now.Year - src.DateOfBirth!.Value.Year,
       srcCond => srcCond.DateOfBirth.HasValue)
   ```

3. **Ignoring Properties**
   ```csharp
   // Using configuration
   .Ignore(dest => dest.PropertyName)

   // Using attribute
   [AdaptIgnore]
   public string PropertyName { get; set; }
   ```

4. **Two-Way Mapping**
   ```csharp
   // Enable bidirectional mapping
   config.NewConfig<Student, StudentResponse>()
       .TwoWays();
   ```

5. **Projection Support**
   ```csharp
   // Efficiently select only needed properties
   var response = dbContext.Students
       .ProjectToType<StudentResponse>()
       .ToList();
   ```

## Advanced Features

1. **Custom Value Transformations**
   ```csharp
   .Map(dest => dest.FullName, 
       src => string.Join(" ", src.FirstName, src.MiddleName, src.LastName)
           .Trim())
   ```

2. **Conditional Mapping**
   ```csharp
   .Map(dest => dest.Status, 
       src => src.Age > 18 ? "Adult" : "Minor",
       srcCond => srcCond.DateOfBirth.HasValue)
   ```

3. **Nested Object Mapping**
   ```csharp
   .Map(dest => dest.DepartmentInfo, src => new
   {
       src.Department.Name,
       src.Department.Code
   })
   ```

4. **Collection Mapping**
   ```csharp
   config.NewConfig<List<Student>, List<StudentResponse>>()
       .Map(dest => dest, src => src.Select(x => x.Adapt<StudentResponse>()));
   ```

These advanced features demonstrate Mapster's flexibility in handling complex mapping scenarios while maintaining clean and maintainable code.
