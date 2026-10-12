# Útmutató: EF Core projekt készítése Visual Studio 2022-ben Parancssor (CLI) Használatával

## 1. Projekt létrehozása, csomagok telepítése és megnyitása parancssorból

A nem megfelelő csomagverziók okozta kompatibilitási hibák elkerülése érdekében a .NET 8-as keretrendszerhez rögzített **8.0.x** verziójú csomagokat használunk.

Nyiss egy **Parancssort (CMD)** vagy **PowerShell** ablakot, majd futtasd az alábbi lépéseket:

### A) Új projekt létrehozása
A tananyagban bemutatott `builder.Services.AddDbContext<AppDbContext>(...)` regisztrációs kódhoz hozzunk létre egy **ASP.NET Core Web API** projektet:

```bash
dotnet new webapi -n DiakProjekt -f net8.0
cd DiakProjekt
```

### B) A globális EF Core CLI eszköz telepítése (megadott verzióval)
Telepítsd az EF Core parancssori eszközt globálisan a számítógépre:

```bash
dotnet tool install --global dotnet-ef --version 8.0.11
```
*(Ha már telepítve van, a `dotnet tool update --global dotnet-ef --version 8.0.11` parancssal frissítheted).*

### C) A szükséges NuGet csomagok telepítése konkrét verziókkal
Telepítsd a projekthez szükséges EF Core csomagokat fix verziószámmal:

1. **EF Core alapcsomag:**
   ```bash
   dotnet add package Microsoft.EntityFrameworkCore -v 8.0.11
   ```
2. **Adatbázis-szolgáltató (Database Provider):**
   - *MySQL használata esetén:*
     ```bash
     dotnet add package MySql.EntityFrameworkCore -v 8.0.8
     ```
   - *SQL Server használata esetén:*
     ```bash
     dotnet add package Microsoft.EntityFrameworkCore.SqlServer -v 8.0.11
     ```
3. **Tervező és migrációs eszközök a projekthez:**
   ```bash
   dotnet add package Microsoft.EntityFrameworkCore.Design -v 8.0.11
   dotnet add package Microsoft.EntityFrameworkCore.Tools -v 8.0.11
   ```

### D) A projekt megnyitása Visual Studio 2022-ben parancssorból
A projekt mappájában állva futtasd az alábbi parancsot, amely megnyitja a projektet a Visual Studio 2022-ben:

```bash
start DiakProjekt.csproj
```
*(Ha megoldás/solution fájllal dolgozol, a `start DiakProjekt.sln` vagy a `devenv DiakProjekt.csproj` parancs is használható).*

---

## 2. Adatbázis-kapcsolat beállítása és regisztráció

1. **Kapcsolati karakterlánc (Connection String) megadása:**
   Az `appsettings.json` fájlban add meg az adatbázis elérhetőségét:
   ```json
   {
     "ConnectionStrings": {
       "DefaultConnection": "Server=localhost;Database=MintaDb;User=root;Password=secret;"
     }
   }
   ```

2. **A DbContext regisztrálása a `Program.cs` fájlban:**
   ```csharp
   var connStr = builder.Configuration.GetConnectionString("DefaultConnection");
   builder.Services.AddDbContext<AppDbContext>(options => options.UseMySQL(connStr));
   ```

---

## 3. Entitások (osztályok) és DataAnnotations beállítása

A Code-First megközelítés lényege, hogy C# osztályokat hozunk létre, amelyeket az EF Core táblákká és oszlopokká képez le. Alapesetben minden `Id` nevű mező elsődleges kulcs (`int` esetén `auto_increment`), a `string` pedig hosszú szövegmező.

### Entitás példák:

```csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

[Table("Diakok")] // Táblanév felülírása
public class Student
{
    [Key] // Elsődleges kulcs
    [DatabaseGenerated(DatabaseGeneratedOption.Identity)] // Adatbázis által generált érték
    public int StudentId { get; set; }

    [Required] // NOT NULL megszorítás az adatbázisban
    [MaxLength(50)] // Karakterlánc maximális hossza
    [Column("KeresztNev", TypeName = "varchar(50)")] // SQL oszlopnév és típus testreszabása
    public string FirstName { get; set; } = null!;

    [NotMapped] // Nem kerül be az adatbázisba
    public string FullName => $"{FirstName}";

    // N:M kapcsolathoz tartozó kapcsolótábla listája
    public List<StudentCourse> StudentCourses { get; set; } = new();
}

public class Course
{
    [Key]
    public int CourseId { get; set; }

    [Required]
    [StringLength(100)]
    public string CourseName { get; set; } = null!;

    public List<StudentCourse> CourseCourses { get; set; } = new();
}

// Kapcsolótábla az N:M kapcsolathoz
public class StudentCourse
{
    [ForeignKey("Student")] // Idegen kulcs
    public int StudentId { get; set; }
    public Student Student { get; set; } = null!;

    [ForeignKey("Course")] // Idegen kulcs
    public int CourseId { get; set; }
    public Course Course { get; set; } = null!;
}
```

---

## 4. Az `AppDbContext` és a Fluent API

A `DbContext` mintaként a **UnitOfWork** (műveletek egyetlen tranzakcióba fogása) és a **Repository** (entitásonkénti gyűjtemény) elveket valósítja meg.

```csharp
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Student> Students { get; set; } = null!;
    public DbSet<Course> Courses { get; set; } = null!;
    public DbSet<StudentCourse> StudentCourses { get; set; } = null!;

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Összetett elsődleges kulcs beállítása a kapcsolótáblához:
        modelBuilder.Entity<StudentCourse>()
            .HasKey(sc => new { sc.StudentId, sc.CourseId });

        // Kapcsolatok konfigurálása Fluent API-val:
        modelBuilder.Entity<StudentCourse>()
            .HasOne(sc => sc.Student)
            .WithMany(s => s.StudentCourses)
            .HasForeignKey(sc => sc.StudentId);

        modelBuilder.Entity<StudentCourse>()
            .HasOne(sc => sc.Course)
            .WithMany(c => c.CourseCourses)
            .HasForeignKey(sc => sc.CourseId);
    }
}
```

*(Megjegyzés: 1:1 kapcsolat esetén `HasOne().WithOne().HasForeignKey<UserProfile>()`, 1:N kapcsolat esetén pedig `HasOne().WithMany().HasForeignKey()` szintaxis használható).*

---

## 5. Migrációk kezelése parancssorból

Futtasd a parancsokat a terminalban/parancssorban a projekt mappájában állva:

* **Új migráció létrehozása:**
  ```bash
  dotnet ef migrations add InitialCreate
  ```
* **Adatbázis frissítése / létrehozása a migrációk alapján:**
  ```bash
  dotnet ef database update
  ```
* **Legutóbbi migráció törlése:**
  ```bash
  dotnet ef migrations remove
  ```
* **Migrációk listázása:**
  ```bash
  dotnet ef migrations list
  ```
* **Erőltetett újragenerálás:**
  ```bash
  dotnet ef migrations add InitialCreate --force
  ```

---

## 6. Adatkezelés: LINQ, Change Tracker és Tranzakciók

### A) LINQ és SQL fordítás
A C#-ban megírt LINQ lekérdezést az EF Core lefordítja SQL utasítássá:
```csharp
// LINQ lekérdezés:
var result = context.Students.Where(s => s.FirstName == "xy").ToList();

// Generált SQL utasítás:
// SELECT ... FROM Diakok WHERE KeresztNev = 'xy'
```

### B) Change Tracker (Változáskövető) állapotok
Beolvasáskor snapshot készül az adatokról, majd a `SaveChanges()` hívásakor összehasonlítás történik:
* **Detached:** Nincs a `DbContext` felügyelete alatt.
* **Unchanged:** Változatlan adat az adatbázisból.
* **Added:** Új entitás (`context.Add()`), mentéskor **`INSERT`** készül.
* **Modified:** Módosult mező, mentéskor **`UPDATE`** készül.
* **Deleted:** Törlésre kijelölve, mentéskor **`DELETE`** készül.

### C) Tranzakciókezelés
* **Implicit tranzakció:** A `context.SaveChanges()` automatikusan egy tranzakcióba csomagolja a műveleteket; hiba esetén rollback történik.
* **Explicit tranzakció:** Manuális tranzakciókezelés:
  ```csharp
  using var transaction = await context.Database.BeginTransactionAsync();
  try
  {
      // Műveletek végrehajtása...
      await context.SaveChangesAsync();
      await transaction.CommitAsync();
  }
  catch (Exception)
  {
      await transaction.RollbackAsync();
  }
  ```
