# FlashLearn - Source Code Structure

This directory contains the complete source code for the FlashLearn microservices application.

## 📁 Project Structure

```
src/
├── Services/              # Microservices
│   ├── Identity/         # Authentication & User Management
│   │   ├── FLC.Identity.Domain
│   │   ├── FLC.Identity.Application
│   │   ├── FLC.Identity.Infrastructure
│   │   └── FLC.Identity.API
│   ├── Decks/            # Deck Management
│   │   ├── FLC.Decks.Domain
│   │   ├── FLC.Decks.Application
│   │   ├── FLC.Decks.Infrastructure
│   │   └── FLC.Decks.API
│   ├── FlashCards/       # FlashCard Management
│   │   ├── FLC.FlashCards.Domain
│   │   ├── FLC.FlashCards.Application
│   │   ├── FLC.FlashCards.Infrastructure
│   │   └── FLC.FlashCards.API
│   └── Categories/       # Category Management (Global)
│       ├── FLC.Categories.Domain
│       ├── FLC.Categories.Application
│       ├── FLC.Categories.Infrastructure
│       └── FLC.Categories.API
├── ApiGateway/           # API Gateway (Ocelot/YARP)
│   └── FLC.Gateway
├── BuildingBlocks/       # Shared Libraries
│   ├── FLC.Shared       # Common DTOs, Enums, Constants
│   ├── FLC.EventBus     # RabbitMQ Event Bus (Optional)
│   └── FLC.Common       # Extensions, Helpers, Utilities
└── Web/                  # Front-End
    └── FLC.FlashLearn.Web  # Angular Application
```

## 🏗️ Architecture

Each microservice follows **Clean Architecture** principles with **Domain-Driven Design (DDD)**:

- **Domain**: Entities, DTOs, Interfaces, Business Rules
- **Application**: Services, ViewModels, AutoMapper Profiles
- **Infrastructure**: DbContext, Repositories, Configurations, Migrations
- **API**: Controllers, Middlewares, Swagger, Dependency Injection

## 🐳 Microservices

### Identity Service
- **Port**: 5001
- **Database**: FlashLearnIdentity
- **Responsibility**: User authentication, JWT generation, ASP.NET Identity management

### Decks Service
- **Port**: 5002
- **Database**: FlashLearnDecks
- **Responsibility**: Deck CRUD operations, deck-category associations

### FlashCards Service
- **Port**: 5003
- **Database**: FlashLearnFlashCards
- **Responsibility**: FlashCard CRUD operations, flashcard-category associations

### Categories Service
- **Port**: 5004
- **Database**: FlashLearnCategories
- **Responsibility**: Global category management

### API Gateway
- **Port**: 5000
- **Technology**: Ocelot or YARP
- **Responsibility**: Routing, authentication, rate limiting, CORS

## 🚀 Getting Started

### Prerequisites
- .NET 8.0 SDK
- Node.js 18+ and npm/yarn
- Docker & Docker Compose
- SQL Server (via Docker)
- Visual Studio 2022 or VS Code

### Running with Docker
```bash
# From the root directory
docker-compose up -d
```

### Running Individually

**Identity Service:**
```bash
cd src/Services/Identity/FLC.Identity.API
dotnet run
```

**Decks Service:**
```bash
cd src/Services/Decks/FLC.Decks.API
dotnet run
```

**FlashCards Service:**
```bash
cd src/Services/FlashCards/FLC.FlashCards.API
dotnet run
```

**Categories Service:**
```bash
cd src/Services/Categories/FLC.Categories.API
dotnet run
```

**API Gateway:**
```bash
cd src/ApiGateway/FLC.Gateway
dotnet run
```

**Angular App:**
```bash
cd src/Web/FLC.FlashLearn.Web
npm install
npm start
```

## 📚 Documentation

- [Business Rules (English)](../BUSINESS_RULES.md)
- [Regras de Negócio (Português)](../REGRAS_DE_NEGOCIO.md)

## 🔑 Key Features

- ✅ ASP.NET Core Identity for user management
- ✅ JWT authentication across all services
- ✅ Clean Architecture + DDD
- ✅ Docker containerization
- ✅ API Gateway pattern
- ✅ Health checks
- ✅ Swagger documentation
- ✅ Entity Framework Core with SQL Server
- ✅ AutoMapper for object mapping
- ✅ Repository pattern
- ✅ Async/await throughout

## 🔐 Security

- JWT tokens with 24-hour expiration
- Password hashing with PBKDF2 (Identity)
- CORS configured
- HTTPS in production
- Secure password requirements

## 📝 Notes

- Each microservice has its own database
- Categories are global (shared across users)
- Decks and FlashCards are user-specific
- Soft delete implemented (Ativo flag)
- All timestamps use UTC

## 🤝 Contributing

Please refer to the business rules documentation before making changes.

## 📄 License

TBD
