# Regras de Negócio - Sistema de Flashcards

## 1. Visão Geral do Sistema

### Stack Tecnológica
- **Front-End**: Angular
- **Back-End**: .NET Core 8.0 (Microserviços)
- **Banco de Dados**: SQL Server
- **ORM**: Entity Framework Core
- **Autenticação**: Identity, JWT (JSON Web Token)
- **Documentação API**: Swagger
- **Containerização**: Docker, Docker Compose
- **API Gateway**: Ocelot ou YARP
- **Message Broker**: RabbitMQ (opcional)
- **Service Discovery**: Consul (opcional)
- **Monitoramento**: Prometheus + Grafana (opcional)

---

## 2. Arquitetura e Camadas

### Arquitetura de Microserviços

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
│       ├── FlashLearn.EventBus (RabbitMQ - opcional)
│       └── FlashLearn.Common (Extensions, Helpers)
├── docker-compose.yml
├── docker-compose.override.yml
└── .dockerignore
```

### Estrutura de Cada Microserviço (Clean Architecture + DDD)

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

## 3. Entidades e Relacionamentos

### 3.1. Entidade: ApplicationUser (Identity)

**Herança:**
```csharp
public class ApplicationUser : IdentityUser<Guid>
```

**Propriedades Herdadas do IdentityUser:**
```csharp
Id: Guid (PK) - herdado
UserName: string - herdado (usado como Nome)
Email: string (obrigatório, max: 256, único) - herdado
EmailConfirmed: bool - herdado
PasswordHash: string - herdado
SecurityStamp: string - herdado
PhoneNumber: string - herdado
PhoneNumberConfirmed: bool - herdado
TwoFactorEnabled: bool - herdado
LockoutEnd: DateTimeOffset? - herdado
LockoutEnabled: bool - herdado
AccessFailedCount: int - herdado
```

**Propriedades Customizadas:**
```csharp
Nome: string (obrigatório, max: 200)
DataCriacao: DateTime
DataAtualizacao: DateTime?
Ativo: bool
```

**Relacionamentos:**
- 1:N com Deck (um usuário pode ter vários decks)
- 1:N com FlashCard (um usuário pode ter vários flashcards)

**Regras:**
- Email deve ser único no sistema (gerenciado pelo Identity)
- Senha validada pelo Identity com PasswordOptions:
  - RequireDigit = true (exige números)
  - RequireLowercase = true (exige minúsculas)
  - RequireUppercase = true (exige maiúsculas)
  - RequireNonAlphanumeric = true (exige caracteres especiais)
  - RequiredLength = 8 (mínimo 8 caracteres)
  - RequiredUniqueChars = 1
- Email deve ser validado (formato válido) pelo Identity
- UserName será preenchido com o Email
- Ao criar usuário, DataCriacao é preenchida automaticamente
- Ativo = true por padrão
- Identity gerencia automaticamente hash de senha com PBKDF2

---

### 3.2. Entidade: Deck

**Propriedades:**
```csharp
Id: Guid (PK)
Nome: string (obrigatório, max: 200)
UsuarioId: Guid (FK, obrigatório)
DataCriacao: DateTime
DataAtualizacao: DateTime?
Ativo: bool
```

**Relacionamentos:**
- N:1 com ApplicationUser (obrigatório)
- N:N com Categoria (opcional)

**Tabela de Junção: DeckCategoria**
```csharp
DeckId: Guid (FK)
CategoriaId: Guid (FK)
DataAssociacao: DateTime
```

**Regras:**
- Nome é obrigatório
- Nome deve ter entre 3 e 200 caracteres
- Um Deck pertence obrigatoriamente a um Usuário
- Um Deck pode ter zero ou várias Categorias
- Ao excluir um Deck, remover todas as associações em DeckCategoria
- Ao excluir um Deck, não excluir as Categorias associadas (apenas a associação)
- Usuário só pode visualizar/editar seus próprios Decks
- Soft delete: ao excluir, apenas marcar Ativo = false

---

### 3.3. Entidade: Categoria

**Propriedades:**
```csharp
Id: Guid (PK)
Nome: string (obrigatório, max: 100)
DataCriacao: DateTime
DataAtualizacao: DateTime?
Ativo: bool
```

**Relacionamentos:**
- N:N com Deck
- N:N com FlashCard

**Tabela de Junção: DeckCategoria**
```csharp
DeckId: Guid (FK)
CategoriaId: Guid (FK)
DataAssociacao: DateTime
```

**Tabela de Junção: FlashCardCategoria**
```csharp
FlashCardId: Guid (FK)
CategoriaId: Guid (FK)
DataAssociacao: DateTime
```

**Regras:**
- Nome é obrigatório
- Nome deve ter entre 2 e 100 caracteres
- Nome deve ser único no sistema (case-insensitive)
- Categorias são globais (compartilhadas entre todos os usuários)
- Ao excluir uma Categoria, remover todas as associações em DeckCategoria e FlashCardCategoria
- Ao excluir uma Categoria, não excluir Decks ou FlashCards associados
- Soft delete: ao excluir, apenas marcar Ativo = false

---

### 3.4. Entidade: FlashCard

**Propriedades:**
```csharp
Id: Guid (PK)
Nome: string (obrigatório, max: 200)
Traducao: string (opcional, max: 200)
DescricaoLinguaEstudada: string (obrigatório, max: 1000)
DescricaoLinguaMaterna: string (obrigatório, max: 1000)
UsuarioId: Guid (FK, obrigatório)
DataCriacao: DateTime
DataAtualizacao: DateTime?
Ativo: bool
```

**Relacionamentos:**
- N:1 com ApplicationUser (obrigatório)
- N:N com Categoria (opcional)

**Tabela de Junção: FlashCardCategoria**
```csharp
FlashCardId: Guid (FK)
CategoriaId: Guid (FK)
DataAssociacao: DateTime
```

**Regras:**
- Nome, DescricaoLinguaEstudada e DescricaoLinguaMaterna são obrigatórios
- Traducao é opcional
- Nome deve ter entre 1 e 200 caracteres
- DescricaoLinguaEstudada e DescricaoLinguaMaterna devem ter entre 1 e 1000 caracteres
- Um FlashCard pertence obrigatoriamente a um Usuário
- Um FlashCard pode ter zero ou várias Categorias
- Ao excluir um FlashCard, remover todas as associações em FlashCardCategoria
- Usuário só pode visualizar/editar seus próprios FlashCards
- Soft delete: ao excluir, apenas marcar Ativo = false

---

## 4. Regras de Negócio por Funcionalidade

### 4.1. Autenticação e Autorização (Identity Framework)

**Registro de Usuário:**
1. Usar UserManager<ApplicationUser>.CreateAsync() para criar usuário
2. Identity valida automaticamente:
   - Se email já existe no sistema
   - Formato do email
   - Força da senha (conforme PasswordOptions configurado)
3. Hash da senha usando Identity (PBKDF2 automático)
4. Preencher propriedades customizadas:
   - Nome (obrigatório)
   - UserName = Email
   - DataCriacao = DateTime.UtcNow
   - Ativo = true
5. Adicionar usuário à role "User" (opcional)
6. Gerar token JWT após criação bem-sucedida
7. Retornar AuthResponseViewModel com token

**Login:**
1. Usar SignInManager<ApplicationUser>.PasswordSignInAsync() ou validação manual
2. Validar se usuário existe (UserManager.FindByEmailAsync)
3. Validar se Ativo = true (propriedade customizada)
4. Validar se email foi confirmado (EmailConfirmed) - opcional
5. Validar se conta não está bloqueada (LockoutEnabled/LockoutEnd)
6. Identity valida senha automaticamente com hash armazenado
7. Gerar token JWT com claims:
   - UserId (Guid)
   - Email
   - Nome (customizado)
   - Role (opcional)
8. Token com validade de 2 horas
9. Refresh token opcional para renovação
10. Registrar data de último acesso (opcional)

**Autorização:**
1. Todos os endpoints (exceto Register e Login) requerem token JWT válido
2. Usar [Authorize] attribute nos controllers
3. Validar token em cada requisição via middleware JWT
4. Extrair UserId do token (User.FindFirstValue(ClaimTypes.NameIdentifier))
5. Filtrar dados do usuário logado automaticamente
6. Suporte a Roles (opcional): [Authorize(Roles = "Admin,User")]

**Configurações do Identity (PasswordOptions):**
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
  AllowedUserNameCharacters (padrão)

SignInOptions:
  RequireConfirmedEmail = false (true em produção)
  RequireConfirmedPhoneNumber = false

LockoutOptions:
  DefaultLockoutTimeSpan = TimeSpan.FromMinutes(5)
  MaxFailedAccessAttempts = 5
  AllowedForNewUsers = true
```

---

### 4.2. CRUD de Deck

**Criar Deck:**
1. Validar autenticação (token JWT)
2. Validar Nome (obrigatório, 3-200 caracteres)
3. Associar UsuarioId do token
4. Categorias são opcionais na criação
5. Se categorias forem informadas:
   - Validar se todas existem e estão ativas
   - Criar associações em DeckCategoria
6. Retornar DeckViewModel com categorias associadas

**Listar Decks:**
1. Validar autenticação
2. Filtrar apenas Decks do usuário logado (UsuarioId = token.UserId)
3. Filtrar apenas Ativo = true
4. Suportar paginação (page, pageSize)
5. Suportar filtro por Nome (busca parcial, case-insensitive)
6. Suportar filtro por CategoriaId (retornar decks que possuem determinada categoria)
7. Incluir lista de Categorias associadas a cada Deck
8. Retornar lista de DeckViewModel

**Obter Deck por Id:**
1. Validar autenticação
2. Validar se Deck existe
3. Validar se Deck pertence ao usuário logado
4. Validar se Ativo = true
5. Incluir lista de Categorias associadas
6. Retornar DeckViewModel

**Atualizar Deck:**
1. Validar autenticação
2. Validar se Deck existe e pertence ao usuário
3. Validar Nome (obrigatório, 3-200 caracteres)
4. Atualizar DataAtualizacao
5. Se categorias forem informadas:
   - Remover todas as associações antigas em DeckCategoria
   - Validar se todas as novas categorias existem e estão ativas
   - Criar novas associações em DeckCategoria
6. Retornar DeckViewModel atualizado

**Excluir Deck:**
1. Validar autenticação
2. Validar se Deck existe e pertence ao usuário
3. Soft delete: marcar Ativo = false
4. Atualizar DataAtualizacao
5. Remover todas as associações em DeckCategoria
6. Não excluir as Categorias associadas

---

### 4.3. CRUD de Categoria

**Criar Categoria:**
1. Validar autenticação (token JWT)
2. Validar Nome (obrigatório, 2-100 caracteres)
3. Validar unicidade do Nome (case-insensitive)
4. Criar com Ativo = true
5. Retornar CategoriaViewModel

**Listar Categorias:**
1. Validar autenticação
2. Listar todas as categorias ativas (Ativo = true)
3. Categorias são globais (não filtrar por usuário)
4. Suportar paginação (page, pageSize)
5. Suportar filtro por Nome (busca parcial, case-insensitive)
6. Ordenar por Nome (alfabética)
7. Retornar lista de CategoriaViewModel

**Obter Categoria por Id:**
1. Validar autenticação
2. Validar se Categoria existe
3. Validar se Ativo = true
4. Retornar CategoriaViewModel

**Atualizar Categoria:**
1. Validar autenticação
2. Validar se Categoria existe
3. Validar Nome (obrigatório, 2-100 caracteres)
4. Validar unicidade do Nome (excluindo a própria categoria)
5. Atualizar DataAtualizacao
6. Retornar CategoriaViewModel atualizado

**Excluir Categoria:**
1. Validar autenticação
2. Validar se Categoria existe
3. Soft delete: marcar Ativo = false
4. Atualizar DataAtualizacao
5. Remover todas as associações em DeckCategoria
6. Remover todas as associações em FlashCardCategoria
7. Não excluir Decks ou FlashCards associados

---

### 4.4. CRUD de FlashCard

**Criar FlashCard:**
1. Validar autenticação (token JWT)
2. Validar Nome (obrigatório, 1-200 caracteres)
3. Validar DescricaoLinguaEstudada (obrigatório, 1-1000 caracteres)
4. Validar DescricaoLinguaMaterna (obrigatório, 1-1000 caracteres)
5. Traducao é opcional (max: 200 caracteres)
6. Associar UsuarioId do token
7. Categorias são opcionais na criação
8. Se categorias forem informadas:
   - Validar se todas existem e estão ativas
   - Criar associações em FlashCardCategoria
9. Retornar FlashCardViewModel com categorias associadas

**Listar FlashCards:**
1. Validar autenticação
2. Filtrar apenas FlashCards do usuário logado (UsuarioId = token.UserId)
3. Filtrar apenas Ativo = true
4. Suportar paginação (page, pageSize)
5. Suportar filtro por Nome (busca parcial, case-insensitive)
6. Suportar filtro por CategoriaId (retornar flashcards que possuem determinada categoria)
7. Incluir lista de Categorias associadas a cada FlashCard
8. Retornar lista de FlashCardViewModel

**Obter FlashCard por Id:**
1. Validar autenticação
2. Validar se FlashCard existe
3. Validar se FlashCard pertence ao usuário logado
4. Validar se Ativo = true
5. Incluir lista de Categorias associadas
6. Retornar FlashCardViewModel

**Atualizar FlashCard:**
1. Validar autenticação
2. Validar se FlashCard existe e pertence ao usuário
3. Validar Nome (obrigatório, 1-200 caracteres)
4. Validar DescricaoLinguaEstudada (obrigatório, 1-1000 caracteres)
5. Validar DescricaoLinguaMaterna (obrigatório, 1-1000 caracteres)
6. Traducao é opcional (max: 200 caracteres)
7. Atualizar DataAtualizacao
8. Se categorias forem informadas:
   - Remover todas as associações antigas em FlashCardCategoria
   - Validar se todas as novas categorias existem e estão ativas
   - Criar novas associações em FlashCardCategoria
9. Retornar FlashCardViewModel atualizado

**Excluir FlashCard:**
1. Validar autenticação
2. Validar se FlashCard existe e pertence ao usuário
3. Soft delete: marcar Ativo = false
4. Atualizar DataAtualizacao
5. Remover todas as associações em FlashCardCategoria
6. Não excluir as Categorias associadas

---

## 5. ViewModels e DTOs

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
CategoriasIds: List<Guid> (opcional)
```

**UpdateDeckViewModel:**
```csharp
Id: Guid
Nome: string
CategoriasIds: List<Guid> (opcional)
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
Traducao: string (opcional)
DescricaoLinguaEstudada: string
DescricaoLinguaMaterna: string
CategoriasIds: List<Guid> (opcional)
```

**UpdateFlashCardViewModel:**
```csharp
Id: Guid
Nome: string
Traducao: string (opcional)
DescricaoLinguaEstudada: string
DescricaoLinguaMaterna: string
CategoriasIds: List<Guid> (opcional)
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

### 5.5. Paginação

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

## 6. Padrões e Boas Práticas

### 6.1. Repository Pattern

**Interfaces (Domain):**
```csharp
IDeckRepository
ICategoriaRepository
IFlashCardRepository
```

**Observação:** Não criar IUsuarioRepository - o Identity Framework fornece UserManager<ApplicationUser> e RoleManager<IdentityRole<Guid>> para gerenciar usuários.

**Métodos Padrão:**
- `Task<T> GetByIdAsync(Guid id)`
- `Task<IEnumerable<T>> GetAllAsync()`
- `Task<T> AddAsync(T entity)`
- `Task<T> UpdateAsync(T entity)`
- `Task DeleteAsync(Guid id)`
- `Task<bool> ExistsAsync(Guid id)`

**Métodos Customizados:**
- `Task<IEnumerable<Deck>> GetDecksByUsuarioIdAsync(Guid usuarioId)`
- `Task<IEnumerable<FlashCard>> GetFlashCardsByUsuarioIdAsync(Guid usuarioId)`
- `Task<Categoria> GetCategoriaByNomeAsync(string nome)`

---

### 6.2. Service Layer

**Responsabilidades:**
1. Validações de negócio
2. Orquestração de repositórios
3. Mapeamento Entity ↔ ViewModel (via AutoMapper)
4. Tratamento de exceções
5. Retorno de ViewModels

**Padrão de Retorno:**
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

**Registros Obrigatórios (Infra.IoC):**

```csharp
// DbContext com Identity
services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(configuration.GetConnectionString("DefaultConnection")));

// Identity Configuration
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
    options.SignIn.RequireConfirmedEmail = false; // true em produção
    options.SignIn.RequireConfirmedPhoneNumber = false;

    // Lockout settings
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(5);
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.AllowedForNewUsers = true;
})
.AddEntityFrameworkStores<ApplicationDbContext>()
.AddDefaultTokenProviders();

// Repositories (NÃO incluir IUsuarioRepository - usar UserManager)
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
    options.RequireHttpsMetadata = false; // true em produção
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

**Configurações Obrigatórias:**

1. **ApplicationUser Configuration:**
   ```csharp
   // Não é necessário configurar PK (Id) - Identity já faz isso
   // Configurar apenas propriedades customizadas
   builder.Property(u => u.Nome)
       .IsRequired()
       .HasMaxLength(200);
   
   builder.Property(u => u.DataCriacao)
       .IsRequired();
   
   builder.Property(u => u.Ativo)
       .IsRequired()
       .HasDefaultValue(true);
   
   // Relacionamentos
   builder.HasMany<Deck>()
       .WithOne()
       .HasForeignKey(d => d.UsuarioId)
       .OnDelete(DeleteBehavior.Restrict);
   
   builder.HasMany<FlashCard>()
       .WithOne()
       .HasForeignKey(f => f.UsuarioId)
       .OnDelete(DeleteBehavior.Restrict);
   ```

2. **Deck Configuration:**
   - Id como PK
   - HasOne com ApplicationUser (via UsuarioId)
   - HasMany com DeckCategoria

3. **Categoria Configuration:**
   - Id como PK
   - Nome único (índice único)
   - HasMany com DeckCategoria e FlashCardCategoria

4. **FlashCard Configuration:**
   - Id como PK
   - HasOne com ApplicationUser (via UsuarioId)
   - HasMany com FlashCardCategoria

5. **DeckCategoria Configuration:**
   - Chave composta (DeckId, CategoriaId)
   - HasOne com Deck e Categoria

6. **FlashCardCategoria Configuration:**
   - Chave composta (FlashCardId, CategoriaId)
   - HasOne com FlashCard e Categoria

**Observação:** O ApplicationDbContext deve herdar de IdentityDbContext<ApplicationUser, IdentityRole<Guid>, Guid>

---

### 6.5. AutoMapper Profiles

**Mapeamentos Obrigatórios:**

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

## 7. Tratamento de Erros

### 7.1. Tipos de Exceções

**ValidationException:**
- Erros de validação de dados
- Status Code: 400 (Bad Request)

**UnauthorizedException:**
- Usuário não autenticado
- Status Code: 401 (Unauthorized)

**ForbiddenException:**
- Usuário autenticado mas sem permissão
- Status Code: 403 (Forbidden)

**NotFoundException:**
- Recurso não encontrado
- Status Code: 404 (Not Found)

**ConflictException:**
- Conflito de dados (ex: email já existe)
- Status Code: 409 (Conflict)

**InternalServerException:**
- Erros não tratados
- Status Code: 500 (Internal Server Error)

---

### 7.2. Response Padrão de Erro

```json
{
  "success": false,
  "message": "Mensagem de erro principal",
  "errors": [
    "Erro específico 1",
    "Erro específico 2"
  ],
  "data": null
}
```

---

## 8. Segurança

### 8.1. JWT Configuration

**Claims Obrigatórias:**
- UserId (Guid)
- Email (string)
- Nome (string)
- Role (string) - futuro: "User", "Admin"

**Configurações:**
- Algoritmo: HS256
- Validade: 24 horas
- Secret Key: armazenada em appsettings.json / variável de ambiente

---

### 8.2. Password Hashing (Identity)

- Identity usa PBKDF2 por padrão (PasswordHasher<TUser>)
- Salt gerado automaticamente pelo Identity
- Iterações configuradas internamente (10.000+ por padrão)
- Não é necessário implementar hashing manualmente
- Identity gerencia verificação e atualização automática de hash

---

### 8.3. CORS

**Permitir:**
- Origem do front-end Angular (localhost durante desenvolvimento)
- Métodos: GET, POST, PUT, DELETE
- Headers: Authorization, Content-Type

---

## 9. Migrations

### 9.1. Criação de Migrations

**Ordem Recomendada:**
1. `Initial_Create_Identity` - Tabelas do Identity (AspNetUsers, AspNetRoles, AspNetUserRoles, etc.) + ApplicationUser customizado
2. `Create_Domain_Tables` - Tabelas do domínio (Deck, Categoria, FlashCard)
3. `Create_Junction_Tables` - Tabelas de junção (DeckCategoria, FlashCardCategoria)
4. `Add_Indexes` - Índices únicos e de performance

**Observações:**
- A primeira migration incluirá automaticamente todas as tabelas do Identity
- Tabelas do Identity geradas:
  - AspNetUsers (usuários + propriedades customizadas)
  - AspNetRoles (roles/perfis)
  - AspNetUserRoles (associação usuário-role)
  - AspNetUserClaims (claims customizadas)
  - AspNetUserLogins (logins externos - Google, Facebook, etc.)
  - AspNetUserTokens (tokens de refresh, confirmação, etc.)
  - AspNetRoleClaims (claims por role)

---

### 9.2. Seed Data

**Categorias Iniciais (Opcional):**
- Verbos
- Substantivos
- Adjetivos
- Expressões
- Vocabulário Básico
- Vocabulário Avançado
- Gramática
- Pronúncia

---

## 10. Testes

### 10.1. Testes Unitários

**Camada de Service:**
- Testar todas as validações de negócio
- Mock de repositórios
- Testar retorno de ViewModels
- Testar tratamento de exceções

---

### 10.2. Testes de Integração

**API:**
- Testar endpoints completos
- Usar banco de dados em memória (InMemory)
- Testar autenticação JWT
- Testar autorização por usuário

---

## 11. Performance

### 11.1. Entity Framework

**Otimizações Obrigatórias:**
1. Usar `.AsNoTracking()` em consultas read-only
2. Usar `.Include()` para eager loading de relacionamentos
3. Evitar N+1 queries
4. Usar paginação em todas as listagens
5. Criar índices em colunas de busca frequente (Email, Nome)

---

### 11.2. Caching (Futuro)

**Candidatos:**
- Lista de Categorias (cache de 1 hora)
- Informações do usuário logado (cache de sessão)

---

## 12. Documentação da API

### 12.1. Swagger

**Configuração:**
- Habilitar Swagger UI
- Documentar todos os endpoints
- Incluir exemplos de request/response
- Configurar autenticação JWT no Swagger

**Grupos de Endpoints:**
1. Authentication (Register, Login)
2. Decks (CRUD completo)
3. Categorias (CRUD completo)
4. FlashCards (CRUD completo)

---

## 13. Front-End Angular

### 13.1. Estrutura de Módulos

**Recomendação:**
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
│   ├── shared/ (componentes compartilhados)
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

### 13.2. Services Angular

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
- Verificar se usuário está autenticado
- Redirecionar para /login se não autenticado
- Aplicar em todas as rotas exceto login e register

---

### 13.4. Interceptors

**JwtInterceptor:**
- Adicionar header `Authorization: Bearer {token}` em todas as requisições
- Tratar erro 401 (redirecionar para login)

**ErrorInterceptor:**
- Capturar erros HTTP
- Exibir mensagens de erro amigáveis
- Logar erros no console

---

## 14. Variáveis de Ambiente

### 14.1. appsettings.json (Back-End)

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=FlashcardsDb;Trusted_Connection=true;TrustServerCertificate=true;"
  },
  "Jwt": {
    "SecretKey": "SUA_SECRET_KEY_AQUI_MINIMO_32_CARACTERES",
    "Issuer": "FlashcardsAPI",
    "Audience": "FlashcardsAngular",
    "ExpiresInHours": 24
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
  apiUrl: 'https://localhost:7001/api'
};
```

---

## 15. Endpoints da API

### 15.1. Authentication

```
POST /api/auth/register
POST /api/auth/login
```

---

### 15.2. Decks

```
GET    /api/decks?page=1&pageSize=10&nome=&categoriaId=
GET    /api/decks/{id}
POST   /api/decks
PUT    /api/decks/{id}
DELETE /api/decks/{id}
```

---

### 15.3. Categorias

```
GET    /api/categorias?page=1&pageSize=10&nome=
GET    /api/categorias/{id}
POST   /api/categorias
PUT    /api/categorias/{id}
DELETE /api/categorias/{id}
```

---

### 15.4. FlashCards

```
GET    /api/flashcards?page=1&pageSize=10&nome=&categoriaId=
GET    /api/flashcards/{id}
POST   /api/flashcards
PUT    /api/flashcards/{id}
DELETE /api/flashcards/{id}
```

---

## 16. Docker e Containerização

### 16.1. Dockerfile para cada Microserviço

**Exemplo de Dockerfile (FlashLearn.Identity.API):**
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
      - IdentityServiceUrl=http://identity-api:80
    ports:
      - "5002:80"
    depends_on:
      - sqlserver
      - identity-api
    networks:
      - flashlearn-network

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
      - IdentityServiceUrl=http://identity-api:80
    ports:
      - "5003:80"
    depends_on:
      - sqlserver
      - identity-api
    networks:
      - flashlearn-network

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

### 16.3. Configuração de API Gateway (Ocelot)

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

### 16.4. Comunicação entre Microserviços

**1. Comunicação Síncrona (HTTP):**
- Usar HttpClient com IHttpClientFactory
- Implementar retry policies com Polly
- Circuit breaker para resiliência

**2. Comunicação Assíncrona (RabbitMQ - Opcional):**
- Eventos de domínio (DeckCreatedEvent, FlashCardCreatedEvent)
- Event Bus para publish/subscribe
- Garantir idempotência nos consumers

**3. Service Discovery:**
- Usar Consul para registro e descoberta de serviços
- Health checks para cada microserviço

---

### 16.5. Comandos Docker

**Build de todos os serviços:**
```bash
docker-compose build
```

**Iniciar todos os serviços:**
```bash
docker-compose up -d
```

**Ver logs de um serviço específico:**
```bash
docker-compose logs -f identity-api
```

**Parar todos os serviços:**
```bash
docker-compose down
```

**Parar e remover volumes:**
```bash
docker-compose down -v
```

**Rebuild e restart de um serviço:**
```bash
docker-compose up -d --build identity-api
```

---

## 17. Roadmap Futuro

### Fase 2 (Funcionalidades Avançadas)
- Sistema de revisão espaçada (Spaced Repetition)
- Estatísticas de aprendizado
- Compartilhamento de Decks entre usuários
- Importação/Exportação de Decks (CSV, JSON)
- Imagens e áudio nos FlashCards
- Modo de estudo (quiz, teste)

### Fase 3 (Melhorias)
- Push notifications
- PWA (Progressive Web App)
- Modo offline
- Temas dark/light
- Internacionalização (i18n)
- Gamificação (pontos, níveis, conquistas)

---

## 18. Checklist de Implementação

### Microserviços (.NET Core 8.0)

#### Identity Service
- [ ] Criar estrutura de camadas (Domain, Application, Infrastructure, API)
- [ ] Configurar Entity Framework Core com Identity
- [ ] Criar entidade ApplicationUser (herda de IdentityUser<Guid>)
- [ ] Configurar ApplicationDbContext (herda de IdentityDbContext)
- [ ] Configurar FluentAPI para ApplicationUser
- [ ] Configurar Identity com PasswordOptions, UserOptions, SignInOptions, LockoutOptions
- [ ] Criar Migrations
- [ ] Implementar AuthService (usar UserManager e SignInManager)
- [ ] Criar Controllers de autenticação
- [ ] Implementar geração de JWT
- [ ] Configurar Swagger com autenticação JWT
- [ ] Criar Dockerfile
- [ ] Implementar Health Checks

#### Decks Service
- [ ] Criar estrutura de camadas
- [ ] Criar entidades de domínio (Deck, DeckCategoria)
- [ ] Configurar DbContext e FluentAPI
- [ ] Criar Migrations
- [ ] Implementar Repository e Service
- [ ] Criar Controllers
- [ ] Configurar validação de JWT
- [ ] Implementar comunicação com Identity Service (validação de usuário)
- [ ] Configurar Swagger
- [ ] Criar Dockerfile
- [ ] Implementar Health Checks

#### FlashCards Service
- [ ] Criar estrutura de camadas
- [ ] Criar entidades de domínio (FlashCard, FlashCardCategoria)
- [ ] Configurar DbContext e FluentAPI
- [ ] Criar Migrations
- [ ] Implementar Repository e Service
- [ ] Criar Controllers
- [ ] Configurar validação de JWT
- [ ] Implementar comunicação com Identity Service
- [ ] Configurar Swagger
- [ ] Criar Dockerfile
- [ ] Implementar Health Checks

#### Categories Service
- [ ] Criar estrutura de camadas
- [ ] Criar entidades de domínio (Categoria)
- [ ] Configurar DbContext e FluentAPI
- [ ] Criar Migrations
- [ ] Implementar Repository e Service
- [ ] Criar Controllers
- [ ] Configurar validação de JWT
- [ ] Configurar Swagger
- [ ] Criar Dockerfile
- [ ] Implementar Health Checks

#### API Gateway
- [ ] Criar projeto de API Gateway (Ocelot ou YARP)
- [ ] Configurar roteamento para todos os microserviços
- [ ] Configurar autenticação JWT no gateway
- [ ] Implementar rate limiting
- [ ] Configurar CORS
- [ ] Criar Dockerfile
- [ ] Implementar Health Checks agregados

#### Infraestrutura e DevOps
- [ ] Criar docker-compose.yml
- [ ] Configurar SQL Server no Docker
- [ ] Configurar RabbitMQ no Docker (opcional)
- [ ] Configurar variáveis de ambiente
- [ ] Criar scripts de inicialização
- [ ] Configurar volumes para persistência de dados
- [ ] Configurar network para comunicação entre containers
- [ ] Implementar logging centralizado (opcional)
- [ ] Configurar Prometheus e Grafana (opcional)

### Front-End (Angular)
- [ ] Criar projeto Angular
- [ ] Configurar estrutura de módulos
- [ ] Criar models (interfaces TypeScript)
- [ ] Implementar AuthService
- [ ] Implementar DeckService
- [ ] Implementar CategoriaService
- [ ] Implementar FlashCardService
- [ ] Criar AuthGuard
- [ ] Criar JwtInterceptor
- [ ] Criar ErrorInterceptor
- [ ] Implementar telas de login/register
- [ ] Implementar CRUD de Decks
- [ ] Implementar CRUD de Categorias
- [ ] Implementar CRUD de FlashCards
- [ ] Implementar paginação
- [ ] Implementar filtros
- [ ] Adicionar validações de formulário
- [ ] Implementar feedback de erro/sucesso
- [ ] Responsividade mobile
- [ ] Testes unitários (Jasmine/Karma)

### DevOps
- [ ] Configurar CI/CD
- [ ] Configurar Docker
- [ ] Preparar ambiente de produção
- [ ] Configurar backup do banco de dados
- [ ] Monitoramento e logs

---

**Data de Criação:** 23/11/2025  
**Última Atualização:** 23/11/2025  
**Versão:** 1.0 (Atualizado para usar ASP.NET Core Identity)  
**Autor:** Sistema de Flashcards
