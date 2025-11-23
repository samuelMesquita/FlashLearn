# Business Rules - Flashcards System

## 1. System Overview

### Technology Stack
- **Front-End**: Angular
- **Back-End**: .NET Core 8.0 (Microservices)
- **Database**: SQL Server
- **ORM**: Entity Framework Core
- **Authentication**: Identity, JWT (JSON Web Token)
- **API Documentation**: Swagger
- **Containerization**: Docker, Docker Compose
- **API Gateway**: Ocelot or YARP
- **Message Broker**: RabbitMQ (optional)
- **Service Discovery**: Consul (optional)
- **Monitoring**: Prometheus + Grafana (optional)

---

## 2. Architecture and Layers

### Microservices Architecture

```
FlashLearn.Solution/
├── src/
│   ├── ApiGateway/
│   │   └── FlashLearn.Gateway (Ocelot/YARP)
│   ├── Services/
│   │   ├── Identity/
│   │   │   ├── FlashLearn.Identity.Domain
│   │   │   ├── FlashLearn.Identity.Application
│   │   │   ├── FlashLearn.Identity.Infrastructure
│   │   │   └── FlashLearn.Identity.API
│   │   ├── Decks/
│   │   │   ├── FlashLearn.Decks.Domain
│   │   │   ├── FlashLearn.Decks.Application
│   │   │   ├── FlashLearn.Decks.Infrastructure
│   │   │   └── FlashLearn.Decks.API
│   │   ├── FlashCards/
│   │   │   ├── FlashLearn.FlashCards.Domain
│   │   │   ├── FlashLearn.FlashCards.Application
│   │   │   ├── FlashLearn.FlashCards.Infrastructure
│   │   │   └── FlashLearn.FlashCards.API
│   │   └── Categories/
│   │       ├── FlashLearn.Categories.Domain
│   │       ├── FlashLearn.Categories.Application
│   │       ├── FlashLearn.Categories.Infrastructure
│   │       └── FlashLearn.Categories.API
│   └── BuildingBlocks/
│       ├── FlashLearn.Shared (DTOs, Enums, Constants)
│       ├── FlashLearn.EventBus (RabbitMQ - optional)
│       └── FlashLearn.Common (Extensions, Helpers)
├── docker-compose.yml
├── docker-compose.override.yml
└── .dockerignore
```

### Structure of Each Microservice (Clean Architecture + DDD)

```
FlashLearn.{Service}.Domain
├── Entities
├── DTOs
├── Enums
├── Interfaces (IRepositories)
└── Validations

FlashLearn.{Service}.Application
├── Services
├── ViewModels
├── AutoMapper Profiles
└── Interfaces (IServices)

FlashLearn.{Service}.Infrastructure
├── Context (DbContext)
├── Repositories
├── Configurations (FluentAPI)
└── Migrations

FlashLearn.{Service}.API
├── Controllers
├── Middlewares
├── Filters
├── Configuration (Swagger, JWT, CORS)
└── Dockerfile
```

---

## 3. Entities and Relationships

### 3.1. Entity: ApplicationUser (Identity)

**Inheritance:**
```csharp
public class ApplicationUser : IdentityUser<Guid>
```

**Inherited Properties from IdentityUser:**
```csharp
Id: Guid (PK) - inherited
UserName: string - inherited (used as Name)
Email: string (required, max: 256, unique) - inherited
EmailConfirmed: bool - inherited
PasswordHash: string - inherited
SecurityStamp: string - inherited
PhoneNumber: string - inherited
PhoneNumberConfirmed: bool - inherited
TwoFactorEnabled: bool - inherited
LockoutEnd: DateTimeOffset? - inherited
LockoutEnabled: bool - inherited
AccessFailedCount: int - inherited
```

**Custom Properties:**
```csharp
Nome: string (required, max: 200)
DataCriacao: DateTime
DataAtualizacao: DateTime?
Ativo: bool
```

**Relationships:**
- 1:N with Deck (one user can have multiple decks)
- 1:N with FlashCard (one user can have multiple flashcards)

**Rules:**
- Email must be unique in the system (managed by Identity)
- Password validated by Identity with PasswordOptions:
  - RequireDigit = true (requires numbers)
  - RequireLowercase = true (requires lowercase)
  - RequireUppercase = true (requires uppercase)
  - RequireNonAlphanumeric = true (requires special characters)
  - RequiredLength = 8 (minimum 8 characters)
  - RequiredUniqueChars = 1
- Email must be validated (valid format) by Identity
- UserName will be filled with Email
- When creating user, DataCriacao is filled automatically
- Ativo = true by default
- Identity automatically manages password hash with PBKDF2

---

### 3.2. Entity: Deck

**Properties:**
```csharp
Id: Guid (PK)
Nome: string (required, max: 200)
UsuarioId: Guid (FK, required)
DataCriacao: DateTime
DataAtualizacao: DateTime?
Ativo: bool
```

**Relationships:**
- N:1 with ApplicationUser (required)
- N:N with Categoria (optional)

**Junction Table: DeckCategoria**
```csharp
DeckId: Guid (FK)
CategoriaId: Guid (FK)
DataAssociacao: DateTime
```

**Rules:**
- Nome is required
- Nome must be between 3 and 200 characters
- A Deck must belong to a User
- A Deck can have zero or multiple Categories
- When deleting a Deck, remove all associations in DeckCategoria
- When deleting a Deck, do not delete associated Categories (only the association)
- User can only view/edit their own Decks
- Soft delete: when deleting, only mark Ativo = false

---

### 3.3. Entity: Categoria

**Properties:**
```csharp
Id: Guid (PK)
Nome: string (required, max: 100)
DataCriacao: DateTime
DataAtualizacao: DateTime?
Ativo: bool
```

**Relationships:**
- N:N with Deck
- N:N with FlashCard

**Junction Table: DeckCategoria**
```csharp
DeckId: Guid (FK)
CategoriaId: Guid (FK)
DataAssociacao: DateTime
```

**Junction Table: FlashCardCategoria**
```csharp
FlashCardId: Guid (FK)
CategoriaId: Guid (FK)
DataAssociacao: DateTime
```

**Rules:**
- Nome is required
- Nome must be between 2 and 100 characters
- Nome must be unique in the system (case-insensitive)
- Categories are global (shared among all users)
- When deleting a Category, remove all associations in DeckCategoria and FlashCardCategoria
- When deleting a Category, do not delete associated Decks or FlashCards
- Soft delete: when deleting, only mark Ativo = false

---

### 3.4. Entity: FlashCard

**Properties:**
```csharp
Id: Guid (PK)
Nome: string (required, max: 200)
Traducao: string (optional, max: 200)
DescricaoLinguaEstudada: string (required, max: 1000)
DescricaoLinguaMaterna: string (required, max: 1000)
UsuarioId: Guid (FK, required)
DataCriacao: DateTime
DataAtualizacao: DateTime?
Ativo: bool
```

**Relationships:**
- N:1 with ApplicationUser (required)
- N:N with Categoria (optional)

**Junction Table: FlashCardCategoria**
```csharp
FlashCardId: Guid (FK)
CategoriaId: Guid (FK)
DataAssociacao: DateTime
```

**Rules:**
- Nome, DescricaoLinguaEstudada, and DescricaoLinguaMaterna are required
- Traducao is optional
- Nome must be between 1 and 200 characters
- DescricaoLinguaEstudada and DescricaoLinguaMaterna must be between 1 and 1000 characters
- A FlashCard must belong to a User
- A FlashCard can have zero or multiple Categories
- When deleting a FlashCard, remove all associations in FlashCardCategoria
- User can only view/edit their own FlashCards
- Soft delete: when deleting, only mark Ativo = false

---

## 4. Business Rules by Functionality

### 4.1. Authentication and Authorization (Identity Framework)

**User Registration:**
1. Use UserManager<ApplicationUser>.CreateAsync() to create user
2. Identity automatically validates:
   - If email already exists in the system
   - Email format
   - Password strength (according to configured PasswordOptions)
3. Password hash using Identity (automatic PBKDF2)
4. Fill custom properties:
   - Nome (required)
   - UserName = Email
   - DataCriacao = DateTime.UtcNow
   - Ativo = true
5. Add user to "User" role (optional)
6. Generate JWT token after successful creation
7. Return AuthResponseViewModel with token

**Login:**
1. Use SignInManager<ApplicationUser>.PasswordSignInAsync() or manual validation
2. Validate if user exists (UserManager.FindByEmailAsync)
3. Validate if Ativo = true (custom property)
4. Validate if email is confirmed (EmailConfirmed) - optional
5. Validate if account is not locked (LockoutEnabled/LockoutEnd)
6. Identity automatically validates password with stored hash
7. Generate JWT token with claims:
   - UserId (Guid)
   - Email
   - Nome (custom)
   - Role (optional)
8. Token with 24-hour validity
9. Optional refresh token for renewal
10. Log last access date (optional)

**Authorization:**
1. All endpoints (except Register and Login) require valid JWT token
2. Use [Authorize] attribute on controllers
3. Validate token in each request via JWT middleware
4. Extract UserId from token (User.FindFirstValue(ClaimTypes.NameIdentifier))
5. Automatically filter logged-in user data
6. Support for Roles (optional): [Authorize(Roles = "Admin,User")]

**Identity Configuration (PasswordOptions):**
```csharp
PasswordOptions:
  RequireDigit = true
  RequireLowercase = true
  RequireUppercase = true
  RequireNonAlphanumeric = true
  RequiredLength = 8
  RequiredUniqueChars = 1

UserOptions:
  RequireUniqueEmail = true
  AllowedUserNameCharacters (default)

SignInOptions:
  RequireConfirmedEmail = false (true in production)
  RequireConfirmedPhoneNumber = false

LockoutOptions:
  DefaultLockoutTimeSpan = TimeSpan.FromMinutes(5)
  MaxFailedAccessAttempts = 5
  AllowedForNewUsers = true
```

---

### 4.2. Deck CRUD

**Create Deck:**
1. Validate authentication (JWT token)
2. Validate Nome (required, 3-200 characters)
3. Associate UsuarioId from token
4. Categories are optional on creation
5. If categories are informed:
   - Validate if all exist and are active
   - Create associations in DeckCategoria
6. Return DeckViewModel with associated categories

**List Decks:**
1. Validate authentication
2. Filter only Decks from logged-in user (UsuarioId = token.UserId)
3. Filter only Ativo = true
4. Support pagination (page, pageSize)
5. Support filter by Nome (partial search, case-insensitive)
6. Support filter by CategoriaId (return decks that have specific category)
7. Include list of Categories associated with each Deck
8. Return list of DeckViewModel

**Get Deck by Id:**
1. Validate authentication
2. Validate if Deck exists
3. Validate if Deck belongs to logged-in user
4. Validate if Ativo = true
5. Include list of associated Categories
6. Return DeckViewModel

**Update Deck:**
1. Validate authentication
2. Validate if Deck exists and belongs to user
3. Validate Nome (required, 3-200 characters)
4. Update DataAtualizacao
5. If categories are informed:
   - Remove all old associations in DeckCategoria
   - Validate if all new categories exist and are active
   - Create new associations in DeckCategoria
6. Return updated DeckViewModel

**Delete Deck:**
1. Validate authentication
2. Validate if Deck exists and belongs to user
3. Soft delete: mark Ativo = false
4. Update DataAtualizacao
5. Remove all associations in DeckCategoria
6. Do not delete associated Categories

---

### 4.3. Categoria CRUD

**Create Categoria:**
1. Validate authentication (JWT token)
2. Validate Nome (required, 2-100 characters)
3. Validate uniqueness of Nome (case-insensitive)
4. Create with Ativo = true
5. Return CategoriaViewModel

**List Categorias:**
1. Validate authentication
2. List all active categories (Ativo = true)
3. Categories are global (do not filter by user)
4. Support pagination (page, pageSize)
5. Support filter by Nome (partial search, case-insensitive)
6. Order by Nome (alphabetical)
7. Return list of CategoriaViewModel

**Get Categoria by Id:**
1. Validate authentication
2. Validate if Categoria exists
3. Validate if Ativo = true
4. Return CategoriaViewModel

**Update Categoria:**
1. Validate authentication
2. Validate if Categoria exists
3. Validate Nome (required, 2-100 characters)
4. Validate uniqueness of Nome (excluding the category itself)
5. Update DataAtualizacao
6. Return updated CategoriaViewModel

**Delete Categoria:**
1. Validate authentication
2. Validate if Categoria exists
3. Soft delete: mark Ativo = false
4. Update DataAtualizacao
5. Remove all associations in DeckCategoria
6. Remove all associations in FlashCardCategoria
7. Do not delete associated Decks or FlashCards

---

### 4.4. FlashCard CRUD

**Create FlashCard:**
1. Validate authentication (JWT token)
2. Validate Nome (required, 1-200 characters)
3. Validate DescricaoLinguaEstudada (required, 1-1000 characters)
4. Validate DescricaoLinguaMaterna (required, 1-1000 characters)
5. Traducao is optional (max: 200 characters)
6. Associate UsuarioId from token
7. Categories are optional on creation
8. If categories are informed:
   - Validate if all exist and are active
   - Create associations in FlashCardCategoria
9. Return FlashCardViewModel with associated categories

**List FlashCards:**
1. Validate authentication
2. Filter only FlashCards from logged-in user (UsuarioId = token.UserId)
3. Filter only Ativo = true
4. Support pagination (page, pageSize)
5. Support filter by Nome (partial search, case-insensitive)
6. Support filter by CategoriaId (return flashcards that have specific category)
7. Include list of Categories associated with each FlashCard
8. Return list of FlashCardViewModel

**Get FlashCard by Id:**
1. Validate authentication
2. Validate if FlashCard exists
3. Validate if FlashCard belongs to logged-in user
4. Validate if Ativo = true
5. Include list of associated Categories
6. Return FlashCardViewModel

**Update FlashCard:**
1. Validate authentication
2. Validate if FlashCard exists and belongs to user
3. Validate Nome (required, 1-200 characters)
4. Validate DescricaoLinguaEstudada (required, 1-1000 characters)
5. Validate DescricaoLinguaMaterna (required, 1-1000 characters)
6. Traducao is optional (max: 200 characters)
7. Update DataAtualizacao
8. If categories are informed:
   - Remove all old associations in FlashCardCategoria
   - Validate if all new categories exist and are active
   - Create new associations in FlashCardCategoria
9. Return updated FlashCardViewModel

**Delete FlashCard:**
1. Validate authentication
2. Validate if FlashCard exists and belongs to user
3. Soft delete: mark Ativo = false
4. Update DataAtualizacao
5. Remove all associations in FlashCardCategoria
6. Do not delete associated Categories

---

## 5. ViewModels and DTOs

### 5.1. Authentication ViewModels

**RegisterViewModel:**
```csharp
Nome: string
Email: string
Senha: string
ConfirmacaoSenha: string
```

**LoginViewModel:**
```csharp
Email: string
Senha: string
```

**AuthResponseViewModel:**
```csharp
Token: string
ExpiresIn: int
Usuario: UsuarioViewModel
```

**UsuarioViewModel:**
```csharp
Id: Guid
Nome: string
Email: string
DataCriacao: DateTime
```

---

### 5.2. Deck ViewModels

**CreateDeckViewModel:**
```csharp
Nome: string
CategoriasIds: List<Guid> (optional)
```

**UpdateDeckViewModel:**
```csharp
Id: Guid
Nome: string
CategoriasIds: List<Guid> (optional)
```

**DeckViewModel:**
```csharp
Id: Guid
Nome: string
Categorias: List<CategoriaSimplificadaViewModel>
DataCriacao: DateTime
DataAtualizacao: DateTime?
```

**CategoriaSimplificadaViewModel:**
```csharp
Id: Guid
Nome: string
```

---

### 5.3. Categoria ViewModels

**CreateCategoriaViewModel:**
```csharp
Nome: string
```

**UpdateCategoriaViewModel:**
```csharp
Id: Guid
Nome: string
```

**CategoriaViewModel:**
```csharp
Id: Guid
Nome: string
DataCriacao: DateTime
DataAtualizacao: DateTime?
```

---

### 5.4. FlashCard ViewModels

**CreateFlashCardViewModel:**
```csharp
Nome: string
Traducao: string (optional)
DescricaoLinguaEstudada: string
DescricaoLinguaMaterna: string
CategoriasIds: List<Guid> (optional)
```

**UpdateFlashCardViewModel:**
```csharp
Id: Guid
Nome: string
Traducao: string (optional)
DescricaoLinguaEstudada: string
DescricaoLinguaMaterna: string
CategoriasIds: List<Guid> (optional)
```

**FlashCardViewModel:**
```csharp
Id: Guid
Nome: string
Traducao: string
DescricaoLinguaEstudada: string
DescricaoLinguaMaterna: string
Categorias: List<CategoriaSimplificadaViewModel>
DataCriacao: DateTime
DataAtualizacao: DateTime?
```

---

### 5.5. Pagination

**PaginatedResultViewModel<T>:**
```csharp
Items: List<T>
TotalItems: int
Page: int
PageSize: int
TotalPages: int
HasPreviousPage: bool
HasNextPage: bool
```

---

## 6. Patterns and Best Practices

### 6.1. Repository Pattern

**Interfaces (Domain):**
```csharp
IDeckRepository
ICategoriaRepository
IFlashCardRepository
```

**Note:** Do not create IUsuarioRepository - Identity Framework provides UserManager<ApplicationUser> and RoleManager<IdentityRole<Guid>> to manage users.

**Standard Methods:**
- `Task<T> GetByIdAsync(Guid id)`
- `Task<IEnumerable<T>> GetAllAsync()`
- `Task<T> AddAsync(T entity)`
- `Task<T> UpdateAsync(T entity)`
- `Task DeleteAsync(Guid id)`
- `Task<bool> ExistsAsync(Guid id)`

**Custom Methods:**
- `Task<IEnumerable<Deck>> GetDecksByUsuarioIdAsync(Guid usuarioId)`
- `Task<IEnumerable<FlashCard>> GetFlashCardsByUsuarioIdAsync(Guid usuarioId)`
- `Task<Categoria> GetCategoriaByNomeAsync(string nome)`

---

### 6.2. Service Layer

**Responsibilities:**
1. Business validations
2. Repository orchestration
3. Entity ↔ ViewModel mapping (via AutoMapper)
4. Exception handling
5. ViewModel return

**Return Pattern:**
```csharp
Task<ResultViewModel<T>>
```

**ResultViewModel:**
```csharp
Success: bool
Message: string
Data: T
Errors: List<string>
```

---

### 6.3. Dependency Injection

**Required Registrations (per Microservice):**

```csharp
// DbContext with Identity (Identity Service only)
services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(configuration.GetConnectionString("DefaultConnection")));

// Identity Configuration (Identity Service only)
services.AddIdentity<ApplicationUser, IdentityRole<Guid>>(options =>
{
    // Password settings
    options.Password.RequireDigit = true;
    options.Password.RequireLowercase = true;
    options.Password.RequireUppercase = true;
    options.Password.RequireNonAlphanumeric = true;
    options.Password.RequiredLength = 8;
    options.Password.RequiredUniqueChars = 1;

    // User settings
    options.User.RequireUniqueEmail = true;

    // SignIn settings
    options.SignIn.RequireConfirmedEmail = false; // true in production
    options.SignIn.RequireConfirmedPhoneNumber = false;

    // Lockout settings
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(5);
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.AllowedForNewUsers = true;
})
.AddEntityFrameworkStores<ApplicationDbContext>()
.AddDefaultTokenProviders();

// Repositories (DO NOT include IUsuarioRepository - use UserManager)
services.AddScoped<IDeckRepository, DeckRepository>();
services.AddScoped<ICategoriaRepository, CategoriaRepository>();
services.AddScoped<IFlashCardRepository, FlashCardRepository>();

// Services
services.AddScoped<IAutenticacaoService, AutenticacaoService>();
services.AddScoped<IDeckService, DeckService>();
services.AddScoped<ICategoriaService, CategoriaService>();
services.AddScoped<IFlashCardService, FlashCardService>();

// AutoMapper
services.AddAutoMapper(typeof(MappingProfile));

// JWT Authentication
var jwtSettings = configuration.GetSection("Jwt");
var secretKey = Encoding.ASCII.GetBytes(jwtSettings["SecretKey"]);

services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.RequireHttpsMetadata = false; // true in production
    options.SaveToken = true;
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(secretKey),
        ValidateIssuer = true,
        ValidIssuer = jwtSettings["Issuer"],
        ValidateAudience = true,
        ValidAudience = jwtSettings["Audience"],
        ValidateLifetime = true,
        ClockSkew = TimeSpan.Zero
    };
});
```

---

### 6.4. FluentAPI Configuration

**Required Configurations:**

1. **ApplicationUser Configuration (Identity Service):**
   ```csharp
   // No need to configure PK (Id) - Identity already does this
   // Configure only custom properties
   builder.Property(u => u.Nome)
       .IsRequired()
       .HasMaxLength(200);
   
   builder.Property(u => u.DataCriacao)
       .IsRequired();
   
   builder.Property(u => u.Ativo)
       .IsRequired()
       .HasDefaultValue(true);
   
   // Relationships (if managing from Identity Service)
   builder.HasMany<Deck>()
       .WithOne()
       .HasForeignKey(d => d.UsuarioId)
       .OnDelete(DeleteBehavior.Restrict);
   
   builder.HasMany<FlashCard>()
       .WithOne()
       .HasForeignKey(f => f.UsuarioId)
       .OnDelete(DeleteBehavior.Restrict);
   ```

2. **Deck Configuration (Decks Service):**
   - Id as PK
   - HasOne with ApplicationUser (via UsuarioId)
   - HasMany with DeckCategoria

3. **Categoria Configuration (Categories Service):**
   - Id as PK
   - Nome unique (unique index)
   - HasMany with DeckCategoria and FlashCardCategoria

4. **FlashCard Configuration (FlashCards Service):**
   - Id as PK
   - HasOne with ApplicationUser (via UsuarioId)
   - HasMany with FlashCardCategoria

5. **DeckCategoria Configuration (Decks Service):**
   - Composite key (DeckId, CategoriaId)
   - HasOne with Deck and Categoria

6. **FlashCardCategoria Configuration (FlashCards Service):**
   - Composite key (FlashCardId, CategoriaId)
   - HasOne with FlashCard and Categoria

**Note:** ApplicationDbContext in Identity Service should inherit from IdentityDbContext<ApplicationUser, IdentityRole<Guid>, Guid>

**Note:** Other services' DbContext should inherit only from DbContext

---

### 6.5. AutoMapper Profiles

**Required Mappings:**

```csharp
// ApplicationUser (Identity)
CreateMap<ApplicationUser, UsuarioViewModel>()
    .ForMember(dest => dest.Email, opt => opt.MapFrom(src => src.Email))
    .ForMember(dest => dest.Nome, opt => opt.MapFrom(src => src.Nome));

CreateMap<RegisterViewModel, ApplicationUser>()
    .ForMember(dest => dest.UserName, opt => opt.MapFrom(src => src.Email))
    .ForMember(dest => dest.Email, opt => opt.MapFrom(src => src.Email))
    .ForMember(dest => dest.Nome, opt => opt.MapFrom(src => src.Nome))
    .ForMember(dest => dest.DataCriacao, opt => opt.MapFrom(src => DateTime.UtcNow))
    .ForMember(dest => dest.Ativo, opt => opt.MapFrom(src => true));

// Deck
CreateMap<Deck, DeckViewModel>()
    .ForMember(dest => dest.Categorias, opt => opt.MapFrom(src => src.DeckCategorias.Select(dc => dc.Categoria)));
CreateMap<CreateDeckViewModel, Deck>();
CreateMap<UpdateDeckViewModel, Deck>();

// Categoria
CreateMap<Categoria, CategoriaViewModel>();
CreateMap<Categoria, CategoriaSimplificadaViewModel>();
CreateMap<CreateCategoriaViewModel, Categoria>();
CreateMap<UpdateCategoriaViewModel, Categoria>();

// FlashCard
CreateMap<FlashCard, FlashCardViewModel>()
    .ForMember(dest => dest.Categorias, opt => opt.MapFrom(src => src.FlashCardCategorias.Select(fc => fc.Categoria)));
CreateMap<CreateFlashCardViewModel, FlashCard>();
CreateMap<UpdateFlashCardViewModel, FlashCard>();
```

---

## 7. Error Handling

### 7.1. Exception Types

**ValidationException:**
- Data validation errors
- Status Code: 400 (Bad Request)

**UnauthorizedException:**
- User not authenticated
- Status Code: 401 (Unauthorized)

**ForbiddenException:**
- User authenticated but without permission
- Status Code: 403 (Forbidden)

**NotFoundException:**
- Resource not found
- Status Code: 404 (Not Found)

**ConflictException:**
- Data conflict (e.g., email already exists)
- Status Code: 409 (Conflict)

**InternalServerException:**
- Unhandled errors
- Status Code: 500 (Internal Server Error)

---

### 7.2. Standard Error Response

```json
{
  "success": false,
  "message": "Main error message",
  "errors": [
    "Specific error 1",
    "Specific error 2"
  ],
  "data": null
}
```

---

## 8. Security

### 8.1. JWT Configuration

**Required Claims:**
- UserId (Guid)
- Email (string)
- Nome (string)
- Role (string) - future: "User", "Admin"

**Settings:**
- Algorithm: HS256
- Validity: 24 hours
- Secret Key: stored in appsettings.json / environment variable

---

### 8.2. Password Hashing (Identity)

- Identity uses PBKDF2 by default (PasswordHasher<TUser>)
- Salt automatically generated by Identity
- Iterations configured internally (10,000+ by default)
- No need to implement hashing manually
- Identity manages verification and automatic hash update

---

### 8.3. CORS

**Allow:**
- Angular front-end origin (localhost during development)
- Methods: GET, POST, PUT, DELETE
- Headers: Authorization, Content-Type

---

## 9. Migrations

### 9.1. Creating Migrations

**Recommended Order:**

**Identity Service:**
1. `Initial_Create_Identity` - Identity tables (AspNetUsers, AspNetRoles, AspNetUserRoles, etc.) + custom ApplicationUser

**Decks Service:**
1. `Initial_Create_Decks` - Domain tables (Deck, DeckCategoria)

**FlashCards Service:**
1. `Initial_Create_FlashCards` - Domain tables (FlashCard, FlashCardCategoria)

**Categories Service:**
1. `Initial_Create_Categories` - Domain tables (Categoria)

**Notes:**
- Identity Service first migration will automatically include all Identity tables
- Generated Identity tables:
  - AspNetUsers (users + custom properties)
  - AspNetRoles (roles/profiles)
  - AspNetUserRoles (user-role association)
  - AspNetUserClaims (custom claims)
  - AspNetUserLogins (external logins - Google, Facebook, etc.)
  - AspNetUserTokens (refresh tokens, confirmation, etc.)
  - AspNetRoleClaims (claims by role)

---

### 9.2. Seed Data

**Initial Categories (Optional):**
- Verbs
- Nouns
- Adjectives
- Expressions
- Basic Vocabulary
- Advanced Vocabulary
- Grammar
- Pronunciation

---

## 10. Testing

### 10.1. Unit Tests

**Service Layer:**
- Test all business validations
- Repository mocking
- Test ViewModel return
- Test exception handling

---

### 10.2. Integration Tests

**API:**
- Test complete endpoints
- Use in-memory database (InMemory)
- Test JWT authentication
- Test authorization per user

---

## 11. Performance

### 11.1. Entity Framework

**Required Optimizations:**
1. Use `.AsNoTracking()` on read-only queries
2. Use `.Include()` for eager loading of relationships
3. Avoid N+1 queries
4. Use pagination in all listings
5. Create indexes on frequently searched columns (Email, Nome)

---

### 11.2. Caching (Future)

**Candidates:**
- Category list (1-hour cache)
- Logged-in user information (session cache)

---

## 12. API Documentation

### 12.1. Swagger

**Configuration:**
- Enable Swagger UI
- Document all endpoints
- Include request/response examples
- Configure JWT authentication in Swagger

**Endpoint Groups:**
1. Authentication (Register, Login)
2. Decks (Full CRUD)
3. Categories (Full CRUD)
4. FlashCards (Full CRUD)

---

## 13. Angular Front-End

### 13.1. Module Structure

**Recommendation:**
```
src/
├── app/
│   ├── core/ (guards, interceptors, services)
│   │   ├── guards/
│   │   │   └── auth.guard.ts
│   │   ├── interceptors/
│   │   │   └── jwt.interceptor.ts
│   │   └── services/
│   │       ├── auth.service.ts
│   │       └── api.service.ts
│   ├── shared/ (shared components)
│   │   ├── components/
│   │   └── models/
│   ├── features/
│   │   ├── auth/
│   │   │   ├── login/
│   │   │   └── register/
│   │   ├── decks/
│   │   │   ├── deck-list/
│   │   │   ├── deck-form/
│   │   │   └── deck-detail/
│   │   ├── categorias/
│   │   │   ├── categoria-list/
│   │   │   └── categoria-form/
│   │   └── flashcards/
│   │       ├── flashcard-list/
│   │       ├── flashcard-form/
│   │       └── flashcard-detail/
│   └── app.component.ts
```

---

### 13.2. Angular Services

**AuthService:**
- login()
- register()
- logout()
- getToken()
- isAuthenticated()
- getCurrentUser()

**DeckService:**
- getDecks(page, pageSize, filters)
- getDeckById(id)
- createDeck(deck)
- updateDeck(id, deck)
- deleteDeck(id)

**CategoriaService:**
- getCategorias(page, pageSize, filters)
- getCategoriaById(id)
- createCategoria(categoria)
- updateCategoria(id, categoria)
- deleteCategoria(id)

**FlashCardService:**
- getFlashCards(page, pageSize, filters)
- getFlashCardById(id)
- createFlashCard(flashcard)
- updateFlashCard(id, flashcard)
- deleteFlashCard(id)

---

### 13.3. Guards

**AuthGuard:**
- Check if user is authenticated
- Redirect to /login if not authenticated
- Apply to all routes except login and register

---

### 13.4. Interceptors

**JwtInterceptor:**
- Add header `Authorization: Bearer {token}` to all requests
- Handle 401 error (redirect to login)

**ErrorInterceptor:**
- Capture HTTP errors
- Display user-friendly error messages
- Log errors to console

---

## 14. Environment Variables

### 14.1. appsettings.json (Back-End)

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=sqlserver;Database=FlashLearn{Service};User Id=sa;Password=YourStrong@Passw0rd;TrustServerCertificate=true;"
  },
  "Jwt": {
    "SecretKey": "YOUR_SECRET_KEY_HERE_MINIMUM_32_CHARACTERS",
    "Issuer": "FlashLearnAPI",
    "Audience": "FlashLearnAngular",
    "ExpiresInHours": 24
  },
  "ServiceUrls": {
    "IdentityService": "http://identity-api:80",
    "DecksService": "http://decks-api:80",
    "FlashCardsService": "http://flashcards-api:80",
    "CategoriesService": "http://categories-api:80"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

---

### 14.2. environment.ts (Front-End)

```typescript
export const environment = {
  production: false,
  apiUrl: 'http://localhost:5000/api' // API Gateway URL
};
```

---

## 15. API Endpoints

### 15.1. Authentication (Identity Service)

```
POST /api/auth/register
POST /api/auth/login
```

---

### 15.2. Decks (Decks Service)

```
GET    /api/decks?page=1&pageSize=10&nome=&categoriaId=
GET    /api/decks/{id}
POST   /api/decks
PUT    /api/decks/{id}
DELETE /api/decks/{id}
```

---

### 15.3. Categories (Categories Service)

```
GET    /api/categorias?page=1&pageSize=10&nome=
GET    /api/categorias/{id}
POST   /api/categorias
PUT    /api/categorias/{id}
DELETE /api/categorias/{id}
```

---

### 15.4. FlashCards (FlashCards Service)

```
GET    /api/flashcards?page=1&pageSize=10&nome=&categoriaId=
GET    /api/flashcards/{id}
POST   /api/flashcards
PUT    /api/flashcards/{id}
DELETE /api/flashcards/{id}
```

---

## 16. Docker and Containerization

### 16.1. Dockerfile for Each Microservice

**Example Dockerfile (FlashLearn.Identity.API):**
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["Services/Identity/FlashLearn.Identity.API/FlashLearn.Identity.API.csproj", "Services/Identity/FlashLearn.Identity.API/"]
COPY ["Services/Identity/FlashLearn.Identity.Application/FlashLearn.Identity.Application.csproj", "Services/Identity/FlashLearn.Identity.Application/"]
COPY ["Services/Identity/FlashLearn.Identity.Domain/FlashLearn.Identity.Domain.csproj", "Services/Identity/FlashLearn.Identity.Domain/"]
COPY ["Services/Identity/FlashLearn.Identity.Infrastructure/FlashLearn.Identity.Infrastructure.csproj", "Services/Identity/FlashLearn.Identity.Infrastructure/"]
RUN dotnet restore "Services/Identity/FlashLearn.Identity.API/FlashLearn.Identity.API.csproj"
COPY . .
WORKDIR "/src/Services/Identity/FlashLearn.Identity.API"
RUN dotnet build "FlashLearn.Identity.API.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "FlashLearn.Identity.API.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "FlashLearn.Identity.API.dll"]
```

---

### 16.2. Docker Compose

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: flashlearn-sqlserver
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourStrong@Passw0rd
      - MSSQL_PID=Express
    ports:
      - "1433:1433"
    volumes:
      - sqlserver-data:/var/opt/mssql
    networks:
      - flashlearn-network

  rabbitmq:
    image: rabbitmq:3-management
    container_name: flashlearn-rabbitmq
    environment:
      - RABBITMQ_DEFAULT_USER=guest
      - RABBITMQ_DEFAULT_PASS=guest
    ports:
      - "5672:5672"
      - "15672:15672"
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    networks:
      - flashlearn-network

  identity-api:
    image: flashlearn/identity-api:latest
    container_name: flashlearn-identity-api
    build:
      context: .
      dockerfile: Services/Identity/FlashLearn.Identity.API/Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=http://+:80
      - ConnectionStrings__DefaultConnection=Server=sqlserver;Database=FlashLearnIdentity;User Id=sa;Password=YourStrong@Passw0rd;TrustServerCertificate=true;
      - Jwt__SecretKey=${JWT_SECRET_KEY}
      - Jwt__Issuer=FlashLearnAPI
      - Jwt__Audience=FlashLearnAngular
    ports:
      - "5001:80"
    depends_on:
      - sqlserver
    networks:
      - flashlearn-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  decks-api:
    image: flashlearn/decks-api:latest
    container_name: flashlearn-decks-api
    build:
      context: .
      dockerfile: Services/Decks/FlashLearn.Decks.API/Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=http://+:80
      - ConnectionStrings__DefaultConnection=Server=sqlserver;Database=FlashLearnDecks;User Id=sa;Password=YourStrong@Passw0rd;TrustServerCertificate=true;
      - Jwt__SecretKey=${JWT_SECRET_KEY}
      - ServiceUrls__IdentityService=http://identity-api:80
    ports:
      - "5002:80"
    depends_on:
      - sqlserver
      - identity-api
    networks:
      - flashlearn-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  flashcards-api:
    image: flashlearn/flashcards-api:latest
    container_name: flashlearn-flashcards-api
    build:
      context: .
      dockerfile: Services/FlashCards/FlashLearn.FlashCards.API/Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=http://+:80
      - ConnectionStrings__DefaultConnection=Server=sqlserver;Database=FlashLearnFlashCards;User Id=sa;Password=YourStrong@Passw0rd;TrustServerCertificate=true;
      - Jwt__SecretKey=${JWT_SECRET_KEY}
      - ServiceUrls__IdentityService=http://identity-api:80
    ports:
      - "5003:80"
    depends_on:
      - sqlserver
      - identity-api
    networks:
      - flashlearn-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  categories-api:
    image: flashlearn/categories-api:latest
    container_name: flashlearn-categories-api
    build:
      context: .
      dockerfile: Services/Categories/FlashLearn.Categories.API/Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=http://+:80
      - ConnectionStrings__DefaultConnection=Server=sqlserver;Database=FlashLearnCategories;User Id=sa;Password=YourStrong@Passw0rd;TrustServerCertificate=true;
      - Jwt__SecretKey=${JWT_SECRET_KEY}
    ports:
      - "5004:80"
    depends_on:
      - sqlserver
    networks:
      - flashlearn-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  api-gateway:
    image: flashlearn/api-gateway:latest
    container_name: flashlearn-api-gateway
    build:
      context: .
      dockerfile: ApiGateway/FlashLearn.Gateway/Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=http://+:80
    ports:
      - "5000:80"
    depends_on:
      - identity-api
      - decks-api
      - flashcards-api
      - categories-api
    networks:
      - flashlearn-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  angular-app:
    image: flashlearn/angular-app:latest
    container_name: flashlearn-angular-app
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "4200:80"
    depends_on:
      - api-gateway
    networks:
      - flashlearn-network

volumes:
  sqlserver-data:
  rabbitmq-data:

networks:
  flashlearn-network:
    driver: bridge
```

---

### 16.3. API Gateway Configuration (Ocelot)

**ocelot.json:**
```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/auth/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "identity-api",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/api/auth/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ]
    },
    {
      "DownstreamPathTemplate": "/api/decks/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "decks-api",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/api/decks/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ],
      "AuthenticationOptions": {
        "AuthenticationProviderKey": "Bearer"
      }
    },
    {
      "DownstreamPathTemplate": "/api/flashcards/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "flashcards-api",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/api/flashcards/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ],
      "AuthenticationOptions": {
        "AuthenticationProviderKey": "Bearer"
      }
    },
    {
      "DownstreamPathTemplate": "/api/categorias/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "categories-api",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/api/categorias/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ],
      "AuthenticationOptions": {
        "AuthenticationProviderKey": "Bearer"
      }
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "http://localhost:5000"
  }
}
```

---

### 16.4. Inter-Microservice Communication

**1. Synchronous Communication (HTTP):**
- Use HttpClient with IHttpClientFactory
- Implement retry policies with Polly
- Circuit breaker for resilience

**Example HttpClient Configuration:**
```csharp
services.AddHttpClient("IdentityService", client =>
{
    client.BaseAddress = new Uri(configuration["ServiceUrls:IdentityService"]);
    client.DefaultRequestHeaders.Add("Accept", "application/json");
})
.AddTransientHttpErrorPolicy(policy =>
    policy.WaitAndRetryAsync(3, _ => TimeSpan.FromSeconds(2)))
.AddTransientHttpErrorPolicy(policy =>
    policy.CircuitBreakerAsync(5, TimeSpan.FromSeconds(30)));
```

**2. Asynchronous Communication (RabbitMQ - Optional):**
- Domain events (DeckCreatedEvent, FlashCardCreatedEvent)
- Event Bus for publish/subscribe
- Ensure idempotency in consumers

**Example Event:**
```csharp
public class DeckCreatedEvent
{
    public Guid DeckId { get; set; }
    public Guid UsuarioId { get; set; }
    public string Nome { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

**3. Service Discovery:**
- Use Consul for service registration and discovery
- Health checks for each microservice
- Load balancing between instances

---

### 16.5. Docker Commands

**Build all services:**
```bash
docker-compose build
```

**Start all services:**
```bash
docker-compose up -d
```

**View logs of a specific service:**
```bash
docker-compose logs -f identity-api
```

**Stop all services:**
```bash
docker-compose down
```

**Stop and remove volumes:**
```bash
docker-compose down -v
```

**Rebuild and restart a service:**
```bash
docker-compose up -d --build identity-api
```

**Scale a service:**
```bash
docker-compose up -d --scale decks-api=3
```

**Execute command in running container:**
```bash
docker-compose exec identity-api bash
```

---

## 17. Roadmap Future

### Phase 2 (Advanced Features)
- Spaced repetition system
- Learning statistics
- Deck sharing between users
- Deck import/export (CSV, JSON)
- Images and audio in FlashCards
- Study mode (quiz, test)

### Phase 3 (Improvements)
- Push notifications
- PWA (Progressive Web App)
- Offline mode
- Dark/light themes
- Internationalization (i18n)
- Gamification (points, levels, achievements)

### Phase 4 (Infrastructure)
- Kubernetes orchestration
- CI/CD pipeline
- Centralized logging (ELK Stack)
- Distributed tracing (Jaeger)
- API versioning
- GraphQL gateway (optional)

---

## 18. Implementation Checklist

### Microservices (.NET Core 8.0)

#### Identity Service
- [ ] Create layer structure (Domain, Application, Infrastructure, API)
- [ ] Configure Entity Framework Core with Identity
- [ ] Create ApplicationUser entity (inherits from IdentityUser<Guid>)
- [ ] Configure ApplicationDbContext (inherits from IdentityDbContext)
- [ ] Configure FluentAPI for ApplicationUser
- [ ] Configure Identity with PasswordOptions, UserOptions, SignInOptions, LockoutOptions
- [ ] Create Migrations
- [ ] Implement AuthService (use UserManager and SignInManager)
- [ ] Create authentication Controllers
- [ ] Implement JWT generation
- [ ] Configure Swagger with JWT authentication
- [ ] Create Dockerfile
- [ ] Implement Health Checks
- [ ] Add logging and monitoring

#### Decks Service
- [ ] Create layer structure
- [ ] Create domain entities (Deck, DeckCategoria)
- [ ] Configure DbContext and FluentAPI
- [ ] Create Migrations
- [ ] Implement Repository and Service
- [ ] Create Controllers
- [ ] Configure JWT validation
- [ ] Implement communication with Identity Service (user validation)
- [ ] Configure Swagger
- [ ] Create Dockerfile
- [ ] Implement Health Checks
- [ ] Add logging and monitoring

#### FlashCards Service
- [ ] Create layer structure
- [ ] Create domain entities (FlashCard, FlashCardCategoria)
- [ ] Configure DbContext and FluentAPI
- [ ] Create Migrations
- [ ] Implement Repository and Service
- [ ] Create Controllers
- [ ] Configure JWT validation
- [ ] Implement communication with Identity Service
- [ ] Configure Swagger
- [ ] Create Dockerfile
- [ ] Implement Health Checks
- [ ] Add logging and monitoring

#### Categories Service
- [ ] Create layer structure
- [ ] Create domain entities (Categoria)
- [ ] Configure DbContext and FluentAPI
- [ ] Create Migrations
- [ ] Implement Repository and Service
- [ ] Create Controllers
- [ ] Configure JWT validation
- [ ] Configure Swagger
- [ ] Create Dockerfile
- [ ] Implement Health Checks
- [ ] Add logging and monitoring

#### API Gateway
- [ ] Create API Gateway project (Ocelot or YARP)
- [ ] Configure routing for all microservices
- [ ] Configure JWT authentication in gateway
- [ ] Implement rate limiting
- [ ] Configure CORS
- [ ] Create Dockerfile
- [ ] Implement aggregated Health Checks
- [ ] Add request/response logging

#### Infrastructure and DevOps
- [ ] Create docker-compose.yml
- [ ] Configure SQL Server in Docker
- [ ] Configure RabbitMQ in Docker (optional)
- [ ] Configure environment variables
- [ ] Create initialization scripts
- [ ] Configure volumes for data persistence
- [ ] Configure network for container communication
- [ ] Implement centralized logging (optional)
- [ ] Configure Prometheus and Grafana (optional)
- [ ] Create CI/CD pipeline
- [ ] Document deployment process

### Front-End (Angular)
- [ ] Create Angular project
- [ ] Configure module structure
- [ ] Create models (TypeScript interfaces)
- [ ] Implement AuthService
- [ ] Implement DeckService
- [ ] Implement CategoriaService
- [ ] Implement FlashCardService
- [ ] Create AuthGuard
- [ ] Create JwtInterceptor
- [ ] Create ErrorInterceptor
- [ ] Implement login/register screens
- [ ] Implement Decks CRUD
- [ ] Implement Categories CRUD
- [ ] Implement FlashCards CRUD
- [ ] Implement pagination
- [ ] Implement filters
- [ ] Add form validations
- [ ] Implement error/success feedback
- [ ] Mobile responsiveness
- [ ] Unit tests (Jasmine/Karma)
- [ ] Create Dockerfile for Angular app
- [ ] Configure nginx for production

### Testing
- [ ] Unit tests for all services
- [ ] Integration tests for APIs
- [ ] E2E tests for critical flows
- [ ] Load testing
- [ ] Security testing
- [ ] Container testing

---

**Creation Date:** November 23, 2025  
**Last Update:** November 23, 2025  
**Version:** 2.0 (Updated for Microservices Architecture and ASP.NET Core Identity)  
**Author:** FlashLearn System
