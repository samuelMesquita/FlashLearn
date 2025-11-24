# FlashLearn 🎓

A modern flashcard learning platform built with microservices architecture, ASP.NET Core 8.0, Angular, and Docker.

## 🌟 Features

- 🔐 **Authentication & Authorization** - ASP.NET Core Identity with JWT
- 📚 **Deck Management** - Create and organize flashcard decks
- 🎴 **FlashCard System** - Study with customizable flashcards
- 🏷️ **Categories** - Global category system for organization
- 🐳 **Docker Support** - Full containerization with Docker Compose
- 🚀 **Microservices Architecture** - Scalable and maintainable
- 📊 **API Gateway** - Centralized routing with Ocelot/YARP
- 📝 **Swagger Documentation** - Interactive API documentation

## 🏗️ Architecture

```
FlashLearn/
├── src/
│   ├── Services/              # Microservices
│   │   ├── Identity/         # Authentication Service
│   │   ├── Decks/            # Deck Management Service
│   │   ├── FlashCards/       # FlashCard Management Service
│   │   └── Categories/       # Category Management Service
│   ├── ApiGateway/           # API Gateway (Ocelot/YARP)
│   ├── BuildingBlocks/       # Shared Libraries
│   └── Web/                  # Angular Front-End
├── docker-compose.yml
├── BUSINESS_RULES.md         # English Documentation
└── REGRAS_DE_NEGOCIO.md     # Portuguese Documentation
```

## 🛠️ Technology Stack

### Back-End
- **.NET 8.0** - Modern, cross-platform framework
- **ASP.NET Core Identity** - User management and authentication
- **Entity Framework Core** - ORM for database operations
- **SQL Server** - Relational database
- **JWT** - Token-based authentication
- **AutoMapper** - Object-to-object mapping
- **Swagger/OpenAPI** - API documentation

### Front-End
- **Angular 17+** - Modern web framework
- **TypeScript** - Type-safe JavaScript
- **RxJS** - Reactive programming

### Infrastructure
- **Docker** - Containerization
- **Docker Compose** - Multi-container orchestration
- **Ocelot/YARP** - API Gateway
- **RabbitMQ** - Message broker (optional)

## 🚀 Quick Start

### Prerequisites

- [.NET 8.0 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)
- [Node.js 18+](https://nodejs.org/)
- [Docker Desktop](https://www.docker.com/products/docker-desktop)
- [Visual Studio 2022](https://visualstudio.microsoft.com/) or [VS Code](https://code.visualstudio.com/)

### Running with Docker (Recommended)

1. **Clone the repository**
   ```bash
   git clone https://github.com/samuelMesquita/FlashLearn.git
   cd FlashLearn
   ```

2. **Set up environment variables**
   ```bash
   cp .env.example .env
   # Edit .env with your configuration
   ```

3. **Start all services**
   ```bash
   docker-compose up -d
   ```

4. **Access the application**
   - API Gateway: http://localhost:5000
   - Identity Service: http://localhost:5001
   - Decks Service: http://localhost:5002
   - FlashCards Service: http://localhost:5003
   - Categories Service: http://localhost:5004
   - Angular App: http://localhost:4200
   - RabbitMQ Management: http://localhost:15672

### Running Locally (Development)

1. **Start SQL Server**
   ```bash
   docker-compose up -d sqlserver
   ```

2. **Run each microservice**
   ```bash
   # Identity Service
   cd src/Services/Identity/FLC.Identity.API
   dotnet run

   # Decks Service
   cd src/Services/Decks/FLC.Decks.API
   dotnet run

   # FlashCards Service
   cd src/Services/FlashCards/FLC.FlashCards.API
   dotnet run

   # Categories Service
   cd src/Services/Categories/FLC.Categories.API
   dotnet run

   # API Gateway
   cd src/ApiGateway/FLC.Gateway
   dotnet run
   ```

3. **Run Angular application**
   ```bash
   cd src/Web/FLC.FlashLearn.Web
   npm install
   npm start
   ```

## 📚 Documentation

- [Business Rules (English)](./BUSINESS_RULES.md) - Complete business rules and technical specifications
- [Regras de Negócio (Português)](./REGRAS_DE_NEGOCIO.md) - Regras de negócio completas em português
- [Source Code Structure](./src/README.md) - Detailed source code organization

## 🔑 API Endpoints

### Authentication (Identity Service)
```
POST /api/auth/register - Register new user
POST /api/auth/login    - User login
```

### Decks (Decks Service)
```
GET    /api/decks           - List all decks
GET    /api/decks/{id}      - Get deck by ID
POST   /api/decks           - Create new deck
PUT    /api/decks/{id}      - Update deck
DELETE /api/decks/{id}      - Delete deck
```

### FlashCards (FlashCards Service)
```
GET    /api/flashcards           - List all flashcards
GET    /api/flashcards/{id}      - Get flashcard by ID
POST   /api/flashcards           - Create new flashcard
PUT    /api/flashcards/{id}      - Update flashcard
DELETE /api/flashcards/{id}      - Delete flashcard
```

### Categories (Categories Service)
```
GET    /api/categorias           - List all categories
GET    /api/categorias/{id}      - Get category by ID
POST   /api/categorias           - Create new category
PUT    /api/categorias/{id}      - Update category
DELETE /api/categorias/{id}      - Delete category
```

## 🐳 Docker Commands

```bash
# Build all services
docker-compose build

# Start all services
docker-compose up -d

# Stop all services
docker-compose down

# View logs
docker-compose logs -f [service-name]

# Restart a service
docker-compose restart [service-name]

# Remove volumes (clean database)
docker-compose down -v
```

## 🧪 Testing

```bash
# Run unit tests
dotnet test

# Run integration tests
dotnet test --filter Category=Integration

# Run specific service tests
cd src/Services/Identity/FLC.Identity.Tests
dotnet test
```

## 📦 Project Structure

Each microservice follows **Clean Architecture** with **Domain-Driven Design**:

- **Domain Layer**: Entities, DTOs, Interfaces, Business Rules
- **Application Layer**: Services, ViewModels, AutoMapper Profiles
- **Infrastructure Layer**: DbContext, Repositories, Configurations
- **API Layer**: Controllers, Middlewares, Configuration

## 🔐 Security

- JWT token-based authentication
- Password hashing with PBKDF2 (ASP.NET Core Identity)
- HTTPS in production
- CORS configuration
- Rate limiting on API Gateway
- Input validation and sanitization

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## 📝 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 👥 Authors

- **Samuel Mesquita** - [@samuelMesquita](https://github.com/samuelMesquita)

## 🙏 Acknowledgments

- ASP.NET Core Team
- Angular Team
- Docker Community
- Open Source Contributors

---

**Made with ❤️ for language learners**