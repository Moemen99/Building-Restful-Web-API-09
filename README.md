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

# Data Validation in .NET Core APIs

## Table of Contents
- [Introduction](#introduction)
- [Built-in Validation Attributes](#built-in-validation-attributes)
- [Common Validation Patterns](#common-validation-patterns)
- [Implementation Examples](#implementation-examples)
- [Best Practices](#best-practices)

## Introduction

Data validation is crucial in APIs for:
- Ensuring data integrity
- Protecting against malicious inputs
- Maintaining application reliability
- Providing clear feedback to clients

```mermaid
graph LR
    A[Client Request] --> B[Frontend Validation]
    B --> C[API Validation]
    C --> D[Database Validation]
    C -- Invalid --> E[400 Bad Request]
    C -- Valid --> F[Process Request]
```

## Built-in Validation Attributes

| Attribute | Description | Example | .NET Version |
|-----------|-------------|---------|--------------|
| `[AllowedValues]` | Specifies allowed values | `[AllowedValues("A", "B", "C")]` | ≥8 |
| `[Base64String]` | Validates Base64 encoding | `[Base64String]` | ≥8 |
| `[Compare]` | Compares two properties | `[Compare("PasswordConfirm")]` | All |
| `[CreditCard]` | Validates credit card numbers | `[CreditCard]` | All |
| `[EmailAddress]` | Validates email format | `[EmailAddress]` | All |
| `[Length]` | Validates length range | `[Length(5, 50)]` | ≥8 |
| `[Phone]` | Validates phone numbers | `[Phone]` | All |
| `[Range]` | Validates numeric range | `[Range(1, 5)]` | All |
| `[Required]` | Makes field mandatory | `[Required]` | All |
| `[RegularExpression]` | Validates against pattern | `[RegularExpression(@"\d+")]` | All |

## Common Validation Patterns

### Regular Expression Patterns
```csharp
public static class ValidationPatterns
{
    public const string Password = @"(?=.*[0-9])(?=.*[A-Z])(?=.*[!@#$%^&*]).{8,}";
    public const string Username = @"^[a-zA-Z0-9._@-]{3,20}$";
    public const string CharactersOnly_Eng = @"^[a-zA-Z_]*$";
    public const string CharactersOnly_Ar = @"^[\u0600-\u065F\u066A-\u06EF\u06FA-\u06FF]*$";
    public const string MobileNumber = @"^01[0,1,2,5]{1}[0-9]{8}$";
    public const string NationalID = @"^[2,3]{1}[0-9]{13}$";
}
```

## Implementation Examples

### Using Records (Modern Approach)
```csharp
public record CreatePollRequest(
    [Required(ErrorMessage = "Title is required")]
    [Length(1, 100, ErrorMessage = "Title must be between 1 and 100 characters")]
    [RegularExpression(@"^[\u0600-\u065F\u066A-\u06EF\u06FA-\u06FF]*$", 
        ErrorMessage = "Only Arabic characters are allowed")]
    string Title,

    [MaxLength(500)]
    string? Description
);
```

### Using Classes (Traditional Approach)
```csharp
public class CreatePollRequest
{
    [Required]
    [StringLength(100, MinimumLength = 1)]
    public string Title { get; init; } = string.Empty;

    [MaxLength(500)]
    public string? Description { get; init; }
}
```

### Controller Implementation
```csharp
[ApiController]
public class PollsController : ControllerBase
{
    [HttpPost]
    public IActionResult Create(CreatePollRequest request)
    {
        // [ApiController] handles validation automatically
        // If validation fails, returns 400 Bad Request
        // If validation passes, continues with the logic
        var poll = request.Adapt<Poll>();
        // ... rest of the implementation
    }
}
```

## Best Practices

1. **Multiple Validation Layers**
   ```mermaid
   graph TD
       A[Client Input] --> B[Frontend Validation]
       B --> C[API Validation]
       C --> D[Service Layer Validation]
       D --> E[Database Constraints]
   ```

2. **Custom Error Messages**
   ```csharp
   public record UserRequest(
       [Required(ErrorMessage = "Username is required")]
       [RegularExpression(@"^[a-zA-Z0-9._@-]{3,20}$", 
           ErrorMessage = "Username must be 3-20 characters and contain only letters, numbers, and ._@-")]
       string Username,

       [Required(ErrorMessage = "Password is required")]
       [RegularExpression(@"(?=.*[0-9])(?=.*[A-Z])(?=.*[!@#$%^&*]).{8,}", 
           ErrorMessage = "Password must contain at least one number, one uppercase letter, one special character, and be at least 8 characters long")]
       string Password
   );
   ```

3. **Nullable Reference Types**
   ```xml
   <PropertyGroup>
     <Nullable>enable</Nullable>
   </PropertyGroup>
   ```

4. **Combining Validation Attributes**
   ```csharp
   public class Product
   {
       [Required]
       [Length(3, 50)]
       [RegularExpression(@"^[a-zA-Z0-9\s-]*$")]
       public string Name { get; set; } = string.Empty;

       [Range(0.01, 10000.00)]
       [Required]
       public decimal Price { get; set; }
   }
   ```

5. **API Controller Benefits**
   - Automatic model validation
   - Automatic 400 Bad Request responses
   - Clear validation error messages
   - No need for explicit ModelState checks

Using proper validation ensures data integrity and provides clear feedback to API consumers while maintaining clean and maintainable code.


# Custom Validation Attributes in .NET Core

## Table of Contents
- [Introduction](#introduction)
- [Creating Custom Validation](#creating-custom-validation)
- [Implementation Example](#implementation-example)
- [Usage and Testing](#usage-and-testing)
- [Best Practices](#best-practices)

## Introduction

Custom validation attributes allow you to implement business-specific validation rules that aren't covered by built-in attributes.

```mermaid
graph TD
    A[Model Property] --> B[Custom Validation Attribute]
    B --> C{Validation Logic}
    C --> D[Valid]
    C --> E[Invalid]
    E --> F[Custom Error Message]
```

## Creating Custom Validation

### MinAge Attribute Implementation
```csharp
using System.ComponentModel.DataAnnotations;

[AttributeUsage(AttributeTargets.Property | AttributeTargets.Field)]
 public class MinAgeAttribute(int minAge) : ValidationAttribute
 {
     private readonly int _minAge = minAge;

     protected override ValidationResult? IsValid(object? value, ValidationContext validationContext)
     {

         if (value is null)
         {
             
                 return ValidationResult.Success;
         }

         if (value is not null)
         {
             var date = (DateTime)value;
             Console.WriteLine(DateTime.Today);
             Console.WriteLine(date.AddYears(_minAge));
             if (DateTime.Today < date.AddYears(_minAge)) 
                 return new ValidationResult($"Invalid {validationContext.DisplayName}, Age Must be more than {_minAge} old ");
         }

         return ValidationResult.Success;
     }
 }
```

### Model Class
```csharp
public class Student
{
    public int Id { get; set; }
    
    public string FirstName { get; set; } = string.Empty;
    
    public string MiddleName { get; set; } = string.Empty;
    
    public string LastName { get; set; } = string.Empty;
    
    [MinAge(18)]
    [Display(Name = "Date of Birth")]
    public DateTime? DateOfBirth { get; set; }
}
```

### Controller Implementation
```csharp
[ApiController]
[Route("api/[controller]")]
public class StudentsController : ControllerBase
{
    [HttpPost("test")]
    public IActionResult Test([FromBody] Student request)
    {
        return Ok("Value accepted");
    }
}
```

## Usage and Testing

### Valid Request
```json
{
    "id": 0,
    "firstName": "string",
    "middleName": "string",
    "lastName": "string",
    "dateOfBirth": "2000-03-29T03:27:40.245Z"
}
```

### Invalid Request
```json
{
    "id": 0,
    "firstName": "string",
    "middleName": "string",
    "lastName": "string",
    "dateOfBirth": "2024-03-29T03:26:55.395Z"
}
```

## Best Practices

1. **Flexible Parameter Design**
   ```csharp
   public class MinAgeAttribute : ValidationAttribute
   {
       private readonly int _minAge;
       private const string DefaultErrorMessage = 
           "Age must be {0} years or older";

       public MinAgeAttribute(int minAge) 
           : base(() => string.Format(DefaultErrorMessage, minAge))
       {
           _minAge = minAge;
       }
   }
   ```

2. **Multiple Validation Rules**
   ```csharp
   public class Person
   {
       [MinAge(18)]
       [Display(Name = "Date of Birth")]
       [Required(ErrorMessage = "Date of Birth is required")]
       public DateTime? DateOfBirth { get; set; }
   }
   ```

3. **Custom Error Messages**
   ```csharp
   [MinAge(18, ErrorMessage = "You must be at least 18 years old")]
   [Display(Name = "Date of Birth")]
   public DateTime? DateOfBirth { get; set; }
   ```

4. **Validation Context Usage**
   ```csharp
   protected override ValidationResult? IsValid(
       object? value, 
       ValidationContext validationContext)
   {
       var displayName = validationContext.DisplayName;
       var objectInstance = validationContext.ObjectInstance;
       var objectType = validationContext.ObjectType;

       // Use context information for more detailed validation
       return ValidationResult.Success;
   }
   ```

5. **Type Safety**
   ```csharp
   protected override ValidationResult? IsValid(
       object? value, 
       ValidationContext validationContext)
   {
       if (value is not DateTime date)
       {
           return new ValidationResult(
               $"{validationContext.DisplayName} must be a valid date");
       }
       // Continue validation
   }
   ```

## Key Features of Custom Validation

1. **Reusability**
   - Create once, use across multiple properties
   - Consistent validation logic
   - Easy maintenance

2. **Flexibility**
   - Parameterized validation rules
   - Custom error messages
   - Support for different data types

3. **Integration**
   - Works with MVC model validation
   - Compatible with API controllers
   - Supports dependency injection

4. **Error Handling**
   - Custom error messages
   - Localization support
   - Context-aware validation

This implementation provides a robust way to implement custom business rules while maintaining clean and reusable code.



# FluentValidation in .NET Core APIs

## Table of Contents
- [Setup and Installation](#setup-and-installation)
- [Basic Implementation](#basic-implementation)
- [Advanced Configuration](#advanced-configuration)
- [Integration Methods](#integration-methods)
- [Best Practices](#best-practices)

## Setup and Installation

### Required NuGet Packages
```xml
<ItemGroup>
    <PackageReference Include="FluentValidation.DependencyInjection" Version="11.x.x" />
    <PackageReference Include="FluentValidation.AspNetCore" Version="11.x.x" />
</ItemGroup>
```

### Project Structure
```
YourApi/
├── Contracts/
│   ├── Requests/
│   │   └── CreatePollRequest.cs
│   └── Validations/
│       └── CreatePollRequestValidator.cs
└── Program.cs
```

## Basic Implementation

### Validator Class
```csharp
using FluentValidation;

public class CreatePollRequestValidator 
    : AbstractValidator<CreatePollRequest>
{
    public CreatePollRequestValidator()
    {
        RuleFor(r => r.Title)
            .NotEmpty()
            .MinimumLength(3)
            .MaximumLength(100);
            
        RuleFor(r => r.Description)
            .MaximumLength(500);
    }
}
```

### Manual Integration (Before Automation)
```csharp
[ApiController]
public class PollsController : ControllerBase
{
    [HttpPost]
    public IActionResult Add(
        [FromBody] CreatePollRequest request,
        [FromServices] IValidator<CreatePollRequest> validator)
    {
        var validationResult = validator.Validate(request);
        
        if (!validationResult.IsValid)
        {
            var modelState = new ModelStateDictionary();
            validationResult.Errors.ForEach(error => 
                modelState.AddModelError(
                    error.PropertyName, 
                    error.ErrorMessage));
                    
            return ValidationProblem(modelState);
        }

        var newPoll = _pollService.Add(request.Adapt<Poll>());
        return CreatedAtAction(
            nameof(Get), 
            new { id = newPoll.Id }, 
            newPoll);
    }
}
```

## Advanced Configuration

### Program.cs Setup
```csharp
// Method 1: Register single validator
builder.Services.AddScoped<IValidator<CreatePollRequest>, 
    CreatePollRequestValidator>();

// Method 2: Register all validators from assembly
builder.Services.AddValidatorsFromAssembly(
    Assembly.GetExecutingAssembly());

// Method 3: Register validators from marker assembly
builder.Services.AddValidatorsFromAssemblyContaining<Program>();

// Enable automatic validation
builder.Services
    .AddFluentValidationAutoValidation()
    .AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());
```

### Clean Controller with Automatic Validation
```csharp
[ApiController]
public class PollsController : ControllerBase
{
    [HttpPost]
    public IActionResult Add([FromBody] CreatePollRequest request)
    {
        var newPoll = _pollService.Add(request.Adapt<Poll>());
        return CreatedAtAction(
            nameof(Get), 
            new { id = newPoll.Id }, 
            newPoll);
    }
}
```

## Integration Methods

```mermaid
graph TD
    A[FluentValidation Integration] --> B[Manual Integration]
    A --> C[Automatic Integration]
    B --> D[Explicit Validation Code]
    B --> E[Service Injection]
    C --> F[AddFluentValidationAutoValidation]
    C --> G[Clean Controller Code]
```

### Registration Options Comparison

| Method | Pros | Cons |
|--------|------|------|
| Single Registration | Clear dependencies | Manual registration for each validator |
| Assembly Scanning | Automatic registration | Might register unnecessary validators |
| Marker Interface | Clean architecture support | Additional interface required |

## Best Practices

1. **Validator Organization**
   ```csharp
   public class CreatePollRequestValidator 
       : AbstractValidator<CreatePollRequest>
   {
       public CreatePollRequestValidator()
       {
           // Group related rules
           ApplyTitleRules();
           ApplyDescriptionRules();
       }

       private void ApplyTitleRules()
       {
           RuleFor(x => x.Title)
               .NotEmpty()
               .MinimumLength(3)
               .MaximumLength(100)
               .WithMessage("Title must be between 3 and 100 characters");
       }

       private void ApplyDescriptionRules()
       {
           RuleFor(x => x.Description)
               .MaximumLength(500)
               .When(x => !string.IsNullOrEmpty(x.Description));
       }
   }
   ```

2. **Custom Error Messages**
   ```csharp
   RuleFor(x => x.Title)
       .NotEmpty()
       .WithMessage("Title is required")
       .MinimumLength(3)
       .WithMessage("Title must be at least 3 characters")
       .MaximumLength(100)
       .WithMessage("Title cannot exceed 100 characters");
   ```

3. **Conditional Validation**
   ```csharp
   RuleFor(x => x.OptionalField)
       .NotEmpty()
       .When(x => x.RequireOptionalField)
       .WithMessage("Field is required when RequireOptionalField is true");
   ```

4. **Complex Validations**
   ```csharp
   RuleFor(x => x)
       .Custom((request, context) =>
       {
           if (ComplexValidationFails(request))
           {
               context.AddFailure("Custom validation failed");
           }
       });
   ```

This implementation provides a clean, maintainable way to handle validation in your API endpoints while keeping the controllers focused on their primary responsibilities.
