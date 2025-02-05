# BVMS Project Structure

## 1. Folder Structure
```plaintext
📦 BVMS-BE
|
|__📂 API.MasterData  
|  |__📂 Controllers // API controllers for this module (e.g., finance-related controllers)  
|    |__ InvoiceController.cs  

|__📂 Core.Business // Business logic layer, organized by module  
|  |__📂 Finance // Finance-related business logic  
|     |__ CreateInvoice.cs  
|     |__ SearchInvoices.cs  
|     |__ ... // Use explicit, verb-first function names following the CQRS pattern  

|__📂 Core.Domain // Entities (database) and DTOs (frontend-related), module-based structure  
|  |__📂 Constants // Global constants  
|  |  |__ FinanceConstants.cs // Enums and constants for finance  
|  |
|  |__📂 FinanceData // Finance-related domain objects  
|     |__📂 Entities // Database entity definitions  
|     |__📂 Dtos // DTOs for business logic and controllers  

|__📂 Core.Infrastructure // Infrastructure and utilities  
|  |__📂 DbContext // Database management  
|  |  |__ DataContext.cs // EF Core DbContext, includes entities and migrations  
|  |
|  |__📂 MessageQueue // RabbitMQ utilities  
|  |__ MappingProfile.cs // Store the auto-mapper to transform between Entities & DTOs  

|__📂 ExternalClients // External system integrations  
|  |__📂 DADesk // Integration logic for DADesk  
|  |  |__ Setup.cs // `Configure` method for module installation
|  |
|  |__📂 BusinessCentral // Integration logic for Business Central 
```

## Note:
- Each project in `ExternalClients` should have a `Setup.cs` file
- `appsettings.json` should contains the new section to hold values for the new External Clients

## 2. Entities Design and Mapping
Entity is the core principal data-structure that is migrated into DBs. Should be treated with care and follow the style of other entities.

## 2.1 Inheritance
- Entity can be inherit from BaseEntity or TrackedEntity.
- They will, by default, have auditable fields that are automatically filled by databases without needing for explicit codings, for example:
```C#
public class BaseEtsEventEntity : TrackedEntity
{
    public decimal? TotalConsumptionTimeInDays { get; set; }
    public decimal? ConsumptionPerDayInMetricTons { get; set; }
    ...
}
```

Then these fields will also be added:
```C#
public class TrackedEntity
{
    public Guid Id { get; set; }
    public DateTime CreatedOn { get; set; }
    public string? CreatedById { get; set; }
    public DateTime? ModifiedOn { get; set; }
    public string? ModifiedById { get; set; }
}
```
## Note:
- Do NOT explicitly put values into these fields because `DbContext` will infer them automatically (from user's token and system datetime).

## 2.2 DTOs Design
- DTOs are used by both Business logics and FE (via controllers)
- DTO can be similar or different from Entities.
- Transformation from DTOs to Entities and vice-versa should be handled by `MappingProfile.cs`
- Usually contains 2 versions: `A Reduced Version` for returning values via Search APIs and a `Detailed Value` for the CRUD operations. For example:

```C#
public class ShipmentReducedDto : BaseDto
{
    public Guid ShipmentId { get; set; }
    public Guid? RequestId { get; set; }
    public Guid? CargoTypeId { get; set; }
    public string? ShipmentCode { get; set; }
}

public class ShipmentDto : BaseDto
{
    public int ShipmentNumber { get; set; }
    public Guid? CargoTypeId { get; set; }
    public Guid? RequestId { get; set; }
    public string? ShipmentName { get; set; }
    public string? ShipmentCode { get; set; }
    public string? ShipmentType { get; set; }
    public List<ShipmentCommissionDto> Commissions { get; set; }
    public List<ShipmentPortDto> Ports { get; set; }
}
```
## Note:
- Sometime it is also useful to separated the Detailed DTOs into `CreateDTO` and `UpdateDTO` if FE is limited on how much data they can input during the Create and Update phases.

## 2.3 Auto Mapper
Then, specify the mapping profile for these entities-dtos

```C#
CreateMap<VoyageCrudDto, VoyageEntity>()
    .ForMember(dest => dest.Id, opt => opt.MapFrom(src => Guid.Empty))
    .ForMember(dest => dest.Shipments, opt => opt.Ignore());
CreateMap<ShipmentDto, ShipmentEntity>();
...
```

With this, only minimal code are needed in the Business logics

```C#
return await query
    .ApplyDynamicSorts(dynamicFilters)
    .Select(x => _mapper.Map<ShipmentReducedDto>(x))
    .ToPagedResultAsync(request.SearchParams.CurrentPage, request.SearchParams.PageSize);
```


## 3. `DbContext` Design
After entities is defined, they should be put into `DbContext.cs` for Database migration

## Step 1: includes the Entities into DbSet
```C#
public class DataContext : IdentityDbContext<UserEntity, RoleEntity, string>
{
    ...

    public DataContext(DbContextOptions options, IHttpContextAccessor accessor) : base(options)
    {
        HttpAccessor = accessor;
    }

    public DbSet<ArchetypeEntity> Archetypes { get; set; }
    public DbSet<OfficeEntity> Offices { get; set; }
    // New Entities Should go here
    ...
```
## Step 2: Naming the table
Should contains the module named as pre-fix, for finance module it is `financedata`
```C#
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);

    modelBuilder.Entity<OfficeEntity>().ToTable("usermanagement_offices");
    modelBuilder.Entity<UserEntity>().ToTable("usermanagement_users");
    ...
    modelBuilder.Entity<EstimateProfitAndLossItemEntity>().ToTable("financedata_profitandlossitems");
    modelBuilder.Entity<QuotationEntity>().ToTable("financedata_quotations");
}
```

## Step 3: Define the key and relationships
For complex relationships and default values, should be explicitly defined.
```C#
modelBuilder.Entity<EstimateProfitAndLossItemEntity>(entity =>
{
    entity.HasOne(e => e.Estimate)
        .WithMany(e => e.ProfitAndLossItems)
        .HasForeignKey(e => e.EstimateId);

    entity.HasOne(e => e.Shipment)
        .WithMany(e => e.ProfitAndLossItems)
        .HasForeignKey(e => e.ShipmentId);

    entity.HasOne(e => e.BusinessPartner)
        .WithMany()
        .HasForeignKey(e => e.BusinessPartnerId);

    entity.Property(e => e.CurrencyCode).HasDefaultValue("USD");
    entity.Property(e => e.UsdExchangeRate).HasDefaultValue(1m);
});
```

## Step 4: Migrations

Commands to create a new Db migrations steps
```shell
dotnet ef migrations add "SomeUniqueMigrationNameHere" -c DataContext
```

Then apply the migration
```shell
dotnet ef database update -c DataContext
```

## Note:
- Once the migrations has been applied, the migration scripts should be merged to dev as soon as possible
- Announce other teams who is also working on migrations to keep these scripts in sync, avoid conflict as much.










