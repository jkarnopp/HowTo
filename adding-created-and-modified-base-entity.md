---
description: >-
  This article shows how to inherit from a base entity to assign default values
  to repeated columns like creat and modified dates.
---

# Adding Created and Modified base entity

The first thing we need to do is create a base entity in our models folder.&#x20;

#### BaseEntity.cs

```csharp
public class BaseEntity
    {
        public DateTime CreatedDate { get; set; }
        public DateTime LastModifiedDate { get; set; }

        [Column("CreatedBy")]
        [Display(Name = "Creator")]
        public string CreatedBy { get; set; }

        [Column("LastModifiedBy")]
        [Display(Name = "Modifier")]
        public string ModifiedBy { get; set; }
    }
```

Now whenever we want to add created and modified columns, we just have to inherit from the BaseEntity. Here is an example:

#### Review.cs

```csharp
public class Review : BaseEntity
    {
        public int ReviewId { get; set; }
        public string Name { get; set; }
        public string Description { get; set; }
    }
```

Next we will have to add the IHttpContectAccessor to the configure services section of our startup.cs (Asp.Net Core 5 or earlier) or the Program.cs (Asp.Net 6.0 or greater)

#### Startup.cs

```csharp
services.AddSingleton<IHttpContextAccessor, HttpContextAccessor>();
```

#### Program.cs

```csharp
builder.Services.AddSingleton<IHttpContextAccessor, HttpContextAccessor>();
```

Next we need to update the constructor for the DbContext. In this example I am using an ApplicationDbContext, but your context should look similar.

#### ApplicationDbContext.cs

```csharp
private readonly IHttpContextAccessor _httpContextAccessor;
        public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options, IHttpContextAccessor httpContextAccessor)
            : base(options)
        {
            _httpContextAccessor = httpContextAccessor;
        }
```

I have my data and entities located in seperate projects, so I use a factory to make the migrations work from my site project. In this case, you will have to update the factory code as well.

```csharp
public class DesignTimeDbContextFactory : IDesignTimeDbContextFactory<ApplicationDbContext>
    {
        public ApplicationDbContext CreateDbContext(string[] args)
        {
            IConfigurationRoot configuration = new ConfigurationBuilder().SetBasePath(Directory.GetCurrentDirectory()).AddJsonFile(@Directory.GetCurrentDirectory() + "/../ProjectName.Site/appsettings.json").Build();
            IHttpContextAccessor httpContextAccessor = new HttpContextAccessor();
            var builder = new DbContextOptionsBuilder<ApplicationDbContext>();
            var connectionString = configuration.GetConnectionString("DefaultConnection");
            builder.UseSqlServer(connectionString);
            return new ApplicationDbContext(builder.Options, httpContextAccessor);
        }
    }
```

Next, we will want to create a private method for updating the values in the columns. We do this by checking if the entity is a BaseEntity, and if it is, we check the state and finally update the values. One important item is when an entity is modified, we have to make sure to set the isModified flag for the CreatedDate and CreatedBy to false or they will be overwritten with default values.&#x20;

```csharp
private void OnBeforeSaving()
        {
            var entries = ChangeTracker.Entries();
            var utcNow = DateTime.UtcNow;
            var userId = _httpContextAccessor.HttpContext?.User.FindFirst(ClaimTypes.Email)?.Value;

            var currentUsername = !string.IsNullOrEmpty(userId)
                ? userId
                : "Anonymous";

            foreach (var entry in entries)
            {
                // for entities that inherit from BaseEntity,
                // set LastModifiedDate/ModifiedBy
                if (entry.Entity is BaseEntity trackable)
                {
                    switch (entry.State)
                    {
                        case EntityState.Modified:
                            // set the updated date to "now"
                            trackable.LastModifiedDate = utcNow;
                            trackable.ModifiedBy = currentUsername;

                            // set the isModified flag to false
                            // so the original values arent overwritten
                            entry.Property("CreatedDate").IsModified = false;
                            entry.Property("CreatedBy").IsModified = false;
                            break;

                        case EntityState.Added:
                            // set CreatedDate and LastModifiedDate to "now"
                            trackable.CreatedDate = utcNow;
                            trackable.LastModifiedDate = utcNow;
                            trackable.CreatedBy = currentUsername;
                            trackable.ModifiedBy = currentUsername;
                            break;
                    }
                }
            }
        }
```

The final step is to override the SaveChanges and SaveChangesAsync methods and have them call the OnBeforeSaving method.

```csharp
public override int SaveChanges(bool acceptAllChangesOnSuccess)
        {
            OnBeforeSaving();
            return base.SaveChanges(acceptAllChangesOnSuccess);
        }

public override async Task<int> SaveChangesAsync(
            bool acceptAllChangesOnSuccess,
            CancellationToken cancellationToken = default(CancellationToken))
        {
            OnBeforeSaving();
            return (await base.SaveChangesAsync(acceptAllChangesOnSuccess,
                cancellationToken));
        }
```

With all these steps complete, you should be able to create a new entity inheriting from the base entity, and when you add your migration, the table will be created with the additional columns needed, and CRUD operations will automatically modify the columns as needed.&#x20;
