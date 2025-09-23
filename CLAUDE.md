# Claude Development Guidelines

This file contains development guidelines, commands, and best practices for the Next Net Shop project to help Claude Code and other developers work effectively.

## Development Environment

### Frontend (Next.js)
- **Package Manager**: Bun
- **Runtime**: Node.js with TypeScript
- **Framework**: Next.js 15 with App Router
- **Styling**: Tailwind CSS + Material-UI
- **State Management**: Redux Toolkit
- **Development Port**: 3000

### Backend (.NET)
- **Framework**: .NET 9 Minimal API
- **ORM**: Entity Framework Core
- **Database**: PostgreSQL
- **Authentication**: JWT Bearer tokens
- **Development Port**: 8080
- **API Documentation**: Swagger (available at /swagger in dev)

## Essential Commands

### Frontend Development
```bash
cd next-frontend

# Install dependencies
bun install

# Start development server with Turbopack
bun dev

# Build for production
bun build

# Start production server
bun start

# Run linting
bun lint

# Type checking
npx tsc --noEmit
```

### Backend Development
```bash
cd net-backend

# Restore dependencies
dotnet restore

# Start development server
dotnet run

# Build the application
dotnet build

# Format code with CSharpier
dotnet csharpier .

# Entity Framework migrations
dotnet ef migrations add <MigrationName>
dotnet ef database update
```

## Project Architecture

### Frontend Structure
```
next-frontend/src/
├── app/                    # Next.js App Router pages
│   ├── (shop)/            # Shop layout group
│   │   ├── (browse)/      # Browse pages (products, categories)
│   │   └── layout.tsx     # Shop layout
│   ├── fonts.ts           # Font configurations
│   └── StoreProvider.tsx  # Redux store provider
├── components/            # Reusable UI components
├── lib/                   # Utility functions and configurations
└── styles/               # Global styles and Tailwind config
```

### Backend Structure
```
net-backend/
├── Data/                  # Entity models and DTOs
│   └── Types/            # Entity definitions
├── Products/             # Product-related endpoints
├── Users/                # User authentication and management
├── Cart/                 # Shopping cart functionality
├── Orders/               # Order processing
├── Categories/           # Category management
├── ConfigureServices.cs  # Service configuration
├── ConfigureApp.cs       # Application configuration
└── Program.cs            # Application entry point
```

## Coding Standards

### Frontend Guidelines
- Use TypeScript for all new code
- Follow Next.js App Router patterns
- Use Tailwind CSS for styling with Material-UI components
- Implement proper error boundaries and loading states
- Use Redux Toolkit for state management
- Follow React Hook patterns and functional components

### Backend Guidelines
- Use Minimal API patterns
- Follow REST API conventions
- Implement proper error handling and status codes
- Use DTOs for API responses
- Apply Entity Framework best practices
- Use dependency injection for services
- Implement proper JWT authentication

### General Standards
- Write self-documenting code with clear variable names
- Add comments for complex business logic
- Follow consistent naming conventions
- Implement proper error handling
- Write unit tests for critical functionality

## Database

### Entity Framework Commands
```bash
# Add new migration
dotnet ef migrations add AddNewFeature

# Update database
dotnet ef database update

# Remove last migration
dotnet ef migrations remove

# Generate SQL script
dotnet ef migrations script
```

### Key Entities
- `User` - User accounts and authentication
- `Product` - Product catalog items
- `Category` - Product categorization
- `SubCategory` - Product sub-categorization
- `CartItem` - Shopping cart items
- `Order` - Customer orders
- `OrderItem` - Items within orders

## API Endpoints

### Authentication
- `POST /api/users/register` - User registration
- `POST /api/users/login` - User login
- `POST /api/users/logout` - User logout

### Products
- `GET /api/products` - Get all products
- `GET /api/products/{id}` - Get product by ID
- `POST /api/products` - Create new product (admin)
- `PUT /api/products/{id}` - Update product (admin)
- `DELETE /api/products/{id}` - Delete product (admin)

### Categories
- `GET /api/categories` - Get all categories
- `GET /api/categories/{id}/subcategories` - Get subcategories

### Cart
- `GET /api/cart` - Get user's cart
- `POST /api/cart/items` - Add item to cart
- `PUT /api/cart/items/{id}` - Update cart item
- `DELETE /api/cart/items/{id}` - Remove cart item

### Orders
- `GET /api/orders` - Get user's orders
- `POST /api/orders` - Create new order
- `GET /api/orders/{id}` - Get order details

## Environment Configuration

### Frontend (.env.local)
```env
NEXT_PUBLIC_API_URL=http://localhost:8080
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

### Backend (appsettings.Development.json)
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=nextnetshop;User Id=postgres;Password=yourpassword;"
  },
  "JwtSettings": {
    "SecretKey": "your-secret-key",
    "Issuer": "NextNetShop",
    "Audience": "NextNetShopUsers"
  }
}
```

## Common Tasks

### Adding a New API Endpoint
1. Create endpoint class in appropriate folder (e.g., `Products/`)
2. Define route and HTTP methods
3. Create DTOs in `Data/Types/`
4. Implement business logic
5. Add authentication if required
6. Update Swagger documentation

### Adding a New React Component
1. Create component in `components/` directory
2. Use TypeScript interfaces for props
3. Implement responsive design with Tailwind
4. Add to component exports if reusable
5. Include proper error handling

### Database Schema Changes
1. Modify entity models in `Data/Types/`
2. Create migration: `dotnet ef migrations add <Name>`
3. Review generated migration
4. Update database: `dotnet ef database update`
5. Update related DTOs and endpoints

## Testing

### Frontend Testing
```bash
# Run tests (when implemented)
bun test

# Type checking
npx tsc --noEmit
```

### Backend Testing
```bash
# Run tests (when implemented)
dotnet test
```

## Deployment

### Frontend (Vercel)
- Build command: `bun build`
- Framework preset: Next.js
- Environment variables configured in Vercel dashboard
- Automatic deployments from Git

### Backend (Fly.io)
- Dockerfile provided in net-backend/
- PostgreSQL database on Fly.io
- Environment variables in fly.toml

## Troubleshooting

### Common Issues
1. **CORS errors**: Check CORS configuration in `ConfigureServices.cs`
2. **Database connection**: Verify connection string in appsettings
3. **JWT authentication**: Check token expiration and secret key
4. **Build errors**: Clear node_modules and reinstall dependencies

### Useful Debug Commands
```bash
# Check running processes
ps aux | grep dotnet
ps aux | grep node

# Check ports in use
netstat -tulpn | grep :3000
netstat -tulpn | grep :8080

# Clear Next.js cache
rm -rf next-frontend/.next

# Clean .NET build
dotnet clean && dotnet restore
```

## Additional Resources

- [Next.js Documentation](https://nextjs.org/docs)
- [.NET Minimal APIs](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis)
- [Entity Framework Core](https://docs.microsoft.com/en-us/ef/core/)
- [Tailwind CSS](https://tailwindcss.com/docs)
- [Material-UI](https://mui.com/getting-started/)