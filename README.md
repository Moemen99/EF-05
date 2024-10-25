# Entity Framework Core Fluent API Configuration Approaches

## Table of Contents
- [Basic DbSet Configuration](#basic-dbset-configuration)
- [OnModelCreating Configuration](#onmodelcreating-configuration)
- [Entity Builder Lambda](#entity-builder-lambda)
- [Separate Configuration Classes](#separate-configuration-classes)
- [Migration Process](#migration-process)

## Basic DbSet Configuration

```csharp
public class CompanyDbContext : DbContext
{
    public DbSet<Department> Departments { get; set; }
}
```

## OnModelCreating Configuration

### Table and Schema Configuration
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Department>()
        .ToTable("Departments", "dbo");  // Can also map to View or Function
}
```

### Complete Entity Configuration Example
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Department>()
        // Primary Key Configuration
        .HasKey(nameof(Department.DeptId))
        .UseIdentityColumn(10, 10);  // Starts at 10, increments by 10

    modelBuilder.Entity<Department>()
        // Property Configurations
        .Property(d => d.Name)
        .IsRequired()
        .HasColumnType("varchar")
        .HasColumnName("DepartmentName")
        .HasMaxLength(50)
        //.HasAnnotation("MaxLength", 50)
        .HasDefaultValue("Test");

    modelBuilder.Entity<Department>()
        // Computed Column
        .Property(d => d.DateOfCreation)
        .HasComputedColumnSql("GETDATE()");
}
```

## Entity Builder Lambda
```csharp
// EF Core 3.1+ Approach
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Department>(e => 
    {
        e.ToTable("Departments", "dbo");
        e.HasKey(d => d.DeptId);
        e.Property(d => d.Name)
            .IsRequired()
            .HasColumnType("varchar")
            .HasMaxLength(50);
        e.Property(d => d.DateOfCreation)
            .HasComputedColumnSql("GETDATE()");
    });
}
```

## Separate Configuration Classes

### Project Structure
```
YourProject/
├── Configuration/
│   ├── DepartmentConfiguration.cs
│   ├── EmployeeConfiguration.cs
│   └── ...
```

### Configuration Class Implementation
```csharp
public class DepartmentConfiguration : IEntityTypeConfiguration<Department>
{
    public void Configure(EntityTypeBuilder<Department> builder)
    {
        builder.ToTable("Departments", "dbo");

        builder.HasKey(d => d.DeptId)
               .UseIdentityColumn(10, 10);

        builder.Property(d => d.Name)
               .IsRequired()
               .HasColumnType("varchar")
               .HasColumnName("DepartmentName")
               .HasMaxLength(50)
               .HasDefaultValue("Test");

        builder.Property(d => d.DateOfCreation)
               .HasComputedColumnSql("GETDATE()");
    }
}
```

### Applying Configurations

```mermaid
graph TD
    A[DbContext] -->|Individual| B[Apply Single Configuration]
    A -->|Bulk| C[Apply All Configurations]
    B -->|ApplyConfiguration| D[Single Entity]
    C -->|ApplyConfigurationsFromAssembly| E[All Entities]
    style A fill:#f9f,stroke:#333
    style B fill:#9f9,stroke:#333
    style C fill:#ff9,stroke:#333
```

#### Individual Configuration
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfiguration(new DepartmentConfiguration());
}
```

#### Bulk Configuration using Reflection
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(CompanyDbContext).Assembly);
}
```

## Migration Process

```mermaid
graph LR
    A[Create Configurations] -->|Add-Migration| B[DepartmentUsingFluentAPIs]
    B -->|Update-Database| C[Apply Changes]
    style A fill:#f9f,stroke:#333
    style B fill:#9f9,stroke:#333
    style C fill:#ff9,stroke:#333
```

### Migration Commands
```powershell
# Create migration
Add-Migration DepartmentUsingFluentAPIs

# Apply changes to database
Update-Database
```

## Best Practices

1. **Configuration Organization**
   - One configuration class per entity
   - Use meaningful class names
   - Group related configurations

2. **Code Maintainability**
   - Prefer separate configuration classes
   - Use bulk configuration when possible
   - Document complex configurations

3. **Property Configuration**
   ```csharp
   builder.Property(x => x.PropertyName)  // Use lambda for type safety
         .HasColumnName("DatabaseColumn") // Be explicit about column names
         .IsRequired()                    // Be explicit about constraints
         .HasColumnType("varchar(50)");   // Be explicit about types
   ```

4. **Migration Strategy**
   - Review migrations before applying
   - Test migrations in development
   - Document significant changes

## Notes
- Separate configuration classes improve maintainability
- Bulk configuration reduces boilerplate code
- Consider using computed columns when appropriate
- Always review generated migrations
- Test configurations thoroughly before deployment


# Entity Framework Core Database Operations and Connection Management

## Table of Contents
- [Database Connection Management](#database-connection-management)
- [Working with DbContext](#working-with-dbcontext)
- [Entity States](#entity-states)
- [Adding Entities](#adding-entities)
- [Best Practices](#best-practices)

## Database Connection Management

### Understanding Managed vs Unmanaged Resources

```mermaid
graph TD
    A[Resources] -->|Managed by CLR| B[C# Objects]
    A -->|Unmanaged| C[Database Connections]
    A -->|Unmanaged| D[File Handles]
    B -->|Automatic| E[Garbage Collection]
    C -->|Manual| F[Dispose Required]
    D -->|Manual| F
    style A fill:#f9f,stroke:#333
    style B fill:#9f9,stroke:#333
    style C fill:#ff9,stroke:#333
```

### Connection Disposal Approaches

#### 1. Traditional Try-Finally
```csharp
CompanyDbContext dbContext = new CompanyDbContext();
try
{
    // CRUD Operations
}
finally
{
    // Close Database Connection
    dbContext.Dispose();
}
```

#### 2. Using Statement (C# Traditional)
```csharp
using (CompanyDbContext dbContext = new CompanyDbContext())
{
    // CRUD Operations
}
```

#### 3. Using Declaration (C# Modern)
```csharp
using CompanyDbContext dbContext = new CompanyDbContext();
// CRUD Operations
// Disposed automatically at end of scope
```

## Entity States

### State Enum Values

| State | Description |
|-------|-------------|
| Detached | No tracking by context |
| Unchanged | Tracked, no changes |
| Added | New entity, pending insert |
| Modified | Existing entity, pending update |
| Deleted | Existing entity, pending deletion |

```mermaid
stateDiagram-v2
    [*] --> Detached
    Detached --> Added: Add to Context
    Added --> Unchanged: SaveChanges
    Unchanged --> Modified: Property Change
    Unchanged --> Deleted: Delete
    Modified --> Unchanged: SaveChanges
    Deleted --> Detached: SaveChanges
```

## Adding Entities

### Example Entities
```csharp
Employee E01 = new Employee() 
{
    Name = "Ahmed",
    Age = 22,
    Salary = 5_000,
    EmailAddress = "ahmed@gmail.com"
};

Employee E02 = new Employee() 
{
    Name = "Yassmin",
    Age = 26,
    Salary = 8_000,
    EmailAddress = "Yassmin@gmail.com"
};
```

### Adding Methods Comparison

| Method | Code Example | Notes |
|--------|--------------|-------|
| DbSet Add | `dbContext.Employees.Add(E01)` | Using DbSet property |
| Generic Set | `dbContext.Set<Employee>().Add(E01)` | When no DbSet defined |
| Context Add | `dbContext.Add(E01)` | EF Core 3.1+ feature |
| State Change | `dbContext.Entry(E01).State = EntityState.Added` | Direct state manipulation |

### Checking Entity State
```csharp
// Check current state
Console.WriteLine(dbContext.Entry(E01).State); // Initially Detached
```

## Best Practices

### 1. Resource Management
```csharp
// ✅ Recommended
using var dbContext = new CompanyDbContext();
// Operations here
// Automatic disposal at end of scope

// ❌ Avoid
var dbContext = new CompanyDbContext();
// Operations without disposal
```

### 2. Entity State Management
```csharp
// Check state before operations
if (dbContext.Entry(entity).State == EntityState.Detached)
{
    dbContext.Add(entity);
}
```

### 3. Bulk Operations
```csharp
// Adding multiple entities
using var dbContext = new CompanyDbContext();
var entities = new List<Employee> { E01, E02 };
dbContext.Employees.AddRange(entities);
```

## Performance Considerations

1. **Connection Management**
   - Use short-lived contexts
   - Dispose properly
   - Consider connection pooling

2. **State Tracking**
   ```csharp
   // Disable tracking for read-only operations
   using var dbContext = new CompanyDbContext();
   var employees = dbContext.Employees
       .AsNoTracking()
       .ToList();
   ```

3. **Bulk Operations**
   - Use AddRange instead of multiple Add calls
   - Consider batch size for large operations

## Common Patterns

### Repository Pattern Example
```csharp
public class EmployeeRepository : IDisposable
{
    private readonly CompanyDbContext _context;

    public EmployeeRepository()
    {
        _context = new CompanyDbContext();
    }

    public void Add(Employee employee)
    {
        _context.Employees.Add(employee);
        _context.SaveChanges();
    }

    public void Dispose()
    {
        _context.Dispose();
    }
}
```

## Notes
- Always dispose of DbContext properly
- Understand entity state for effective tracking
- Use appropriate adding method based on scenario
- Consider performance implications of tracking
- Use modern C# features for cleaner code


# Entity Framework Core Change Tracking and SaveChanges Operation

## Table of Contents
- [Change Tracking Process](#change-tracking-process)
- [SaveChanges Operation](#savechanges-operation)
- [Entity State Lifecycle](#entity-state-lifecycle)
- [Best Practices](#best-practices)

## Change Tracking Process

```mermaid
graph TD
    A[Entity Added to Context] -->|State: Added| B[Change Tracker]
    B -->|Tracks Changes| C[SaveChanges Called]
    C -->|Generates SQL| D[Database Update]
    D -->|Updates Entity State| E[State: Unchanged]
    style A fill:#f9f,stroke:#333
    style B fill:#9f9,stroke:#333
    style C fill:#ff9,stroke:#333
    style D fill:#9ff,stroke:#333
```

## SaveChanges Operation

### Process Flow
1. Add entities to context
```csharp
Employee E01 = new Employee() { Name = "Ahmed", Age = 22 };
Employee E02 = new Employee() { Name = "Yassmin", Age = 26 };

dbContext.Employees.Add(E01);
dbContext.Employees.Add(E02);

// State after adding
Console.WriteLine(dbContext.Entry(E01).State); // Added
```

2. Save changes to database
```csharp
dbContext.SaveChanges(); // Generates and executes SQL statements

// State after saving
Console.WriteLine(dbContext.Entry(E01).State); // Unchanged
```

3. Check generated IDs
```csharp
Console.WriteLine($"Employee01 Id = {E01.Id}"); // 1
Console.WriteLine($"Employee02 Id = {E02.Id}"); // 2
```

## Entity State Lifecycle

| Operation Phase | Entity State | Description |
|----------------|--------------|-------------|
| Initial Creation | Detached | Entity not tracked by context |
| After Add to Context | Added | Pending insert to database |
| After SaveChanges | Unchanged | Successfully persisted |
| After Modification | Modified | Pending update to database |
| After Deletion | Deleted | Pending deletion from database |

### State Transitions
```mermaid
stateDiagram-v2
    [*] --> Detached: Create Entity
    Detached --> Added: Add to Context
    Added --> Unchanged: SaveChanges
    Unchanged --> Modified: Modify Entity
    Modified --> Unchanged: SaveChanges
    Unchanged --> Deleted: Delete Entity
    Deleted --> Detached: SaveChanges
```

## Best Practices

### 1. Batch Operations
```csharp
// Perform multiple operations
dbContext.Employees.Add(E01);
dbContext.Employees.Add(E02);
// More operations...

// Single SaveChanges call at the end
dbContext.SaveChanges(); // One database round trip
```

### 2. Change Tracking Management
```csharp
// Check state before operations
if (dbContext.Entry(entity).State == EntityState.Added)
{
    // Handle newly added entity
}
```

### 3. Error Handling
```csharp
try
{
    dbContext.SaveChanges();
}
catch (DbUpdateException ex)
{
    // Handle database update errors
}
```

## Key Points

1. **Change Tracker**
   - Monitors entity states
   - Records all pending changes
   - Optimizes database operations

2. **SaveChanges Benefits**
   - Batches multiple operations
   - Single database round trip
   - Maintains data consistency
   - Updates entity states automatically

3. **State Management**
   ```csharp
   // Before SaveChanges
   dbContext.Employees.Add(entity);     // State: Added
   dbContext.SaveChanges();             // Executes SQL
   // After SaveChanges                 // State: Unchanged
   ```

## Performance Considerations

1. **Batch Size**
   - Group related changes
   - Avoid too many pending changes
   - Consider memory usage

2. **Transaction Scope**
   ```csharp
   using var transaction = dbContext.Database.BeginTransaction();
   try
   {
       // Multiple operations
       dbContext.SaveChanges();
       transaction.Commit();
   }
   catch
   {
       transaction.Rollback();
   }
   ```

## Notes
- SaveChanges generates appropriate SQL statements
- Identity columns are populated after SaveChanges
- Entity state changes to Unchanged after successful save
- Multiple operations can be batched before SaveChanges
- Consider using transactions for complex operations



# Entity Framework Core CRUD Operations and Change Tracking

## Table of Contents
- [Querying Entities](#querying-entities)
- [Change Tracking Behavior](#change-tracking-behavior)
- [Update Operations](#update-operations)
- [Delete Operations](#delete-operations)
- [Best Practices](#best-practices)

## Querying Entities

### Basic Query Syntax
```csharp
// LINQ query returning IQueryable<Employee>
var employee = from E in dbContext.Employee
               where E.EmpId == 1
               select E;

// Get single result
var result = employee.FirstOrDefault();

// Null-safe access
Console.WriteLine(result?.Name ?? "NA");
```

## Change Tracking Behavior

### Default Tracking Configuration
```csharp
// Default behavior - tracks all entities
dbContext.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.TrackAll;

// Disable tracking globally
dbContext.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
```

### Query-Level Tracking Control
```csharp
// No-tracking query for read-only operations
var employee = (from E in dbContext.Employee
                where E.EmpId == 1
                select E).AsNoTracking().FirstOrDefault();
```

## State Transitions in CRUD Operations

```mermaid
stateDiagram-v2
    [*] --> Unchanged: Query Entity
    Unchanged --> Modified: Update Property
    Modified --> Unchanged: SaveChanges
    Unchanged --> Deleted: Remove Entity
    Deleted --> Detached: SaveChanges
```

## CRUD Operations Example

### Update Operation
```csharp
// Query entity
var employee = dbContext.Employee.FirstOrDefault(e => e.EmpId == 1);

// Check initial state
Console.WriteLine(dbContext.Entry(employee).State); // Unchanged

// Modify entity
employee.Name = "Hamada";

// Check state after modification
Console.WriteLine(dbContext.Entry(employee).State); // Modified

// Save changes
dbContext.SaveChanges();

// Check state after save
Console.WriteLine(dbContext.Entry(employee).State); // Unchanged
```

### Delete Operation
```csharp
// Remove entity
dbContext.Employee.Remove(employee);

// Check state after removal
Console.WriteLine(dbContext.Entry(employee).State); // Deleted

// Save changes
dbContext.SaveChanges();

// Check state after save
Console.WriteLine(dbContext.Entry(employee).State); // Detached
```

## State Transitions Table

| Operation | Initial State | Action | New State | After SaveChanges |
|-----------|--------------|--------|------------|-------------------|
| Query | N/A | Load Entity | Unchanged | Unchanged |
| Update | Unchanged | Modify Property | Modified | Unchanged |
| Delete | Unchanged | Remove Entity | Deleted | Detached |

## Best Practices

### 1. Query Optimization
```csharp
// Use no-tracking for read-only queries
var results = dbContext.Employee
    .AsNoTracking()
    .Where(e => e.Age > 25)
    .ToList();
```

### 2. State Verification
```csharp
// Check state before operations
if (dbContext.Entry(employee).State == EntityState.Unchanged)
{
    // Safe to modify
    employee.Name = "New Name";
}
```

### 3. Batch Operations
```csharp
// Perform multiple operations
void UpdateEmployees(List<Employee> employees)
{
    foreach (var emp in employees)
    {
        emp.Salary *= 1.1m; // 10% raise
    }
    dbContext.SaveChanges(); // Single save for all changes
}
```

## Performance Considerations

```mermaid
graph TD
    A[Query Design] -->|Performance Impact| B[Tracking Behavior]
    B -->|Choose| C[TrackAll]
    B -->|Choose| D[NoTracking]
    C -->|Use When| E[Need to Update]
    D -->|Use When| F[Read-Only]
    style A fill:#f9f,stroke:#333
    style B fill:#9f9,stroke:#333
    style C fill:#ff9,stroke:#333
    style D fill:#ff9,stroke:#333
```

### Tracking Considerations
1. **Use Tracking When**
   - Entities need to be updated
   - Change tracking is required
   - Relationships need to be managed

2. **Use No-Tracking When**
   - Read-only operations
   - Better performance needed
   - Memory optimization required

## Notes
- Change tracking affects performance and memory usage
- Consider no-tracking for read-only queries
- Verify entity state before operations
- SaveChanges affects all tracked entities
- State transitions follow a predictable pattern
- Always check for null when querying single entities
