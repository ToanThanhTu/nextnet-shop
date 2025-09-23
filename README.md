# Next Net Shop

A modern e-commerce platform built with Next.js frontend and .NET backend, featuring a full-stack shopping experience with user authentication, product management, and order processing.

## Architecture

- **Frontend**: Next.js 15 with TypeScript, Tailwind CSS, Material-UI, and Redux Toolkit
- **Backend**: .NET 9 Minimal API with Entity Framework Core
- **Database**: PostgreSQL
- **Authentication**: JWT-based authentication
- **Package Manager**: Bun (frontend), NuGet (backend)
- **Deployment**: Frontend on Vercel, Backend on Fly.io

## Features

- 🛒 Shopping cart functionality
- 👤 User authentication and registration
- 📦 Product catalog with categories and subcategories
- 📋 Order management and tracking
- 🔍 Product search and filtering
- 📱 Responsive design
- 🎨 Modern UI with Material-UI and Tailwind CSS

## Project Structure

```
nextnet-shop/
├── next-frontend/          # Next.js frontend application
│   ├── src/               # Source code
│   ├── public/            # Static assets
│   └── package.json       # Frontend dependencies
├── net-backend/           # .NET backend API
│   ├── Data/              # Entity models and DTOs
│   ├── Products/          # Product endpoints
│   ├── Users/             # User authentication
│   ├── Cart/              # Shopping cart endpoints
│   ├── Orders/            # Order management
│   └── Categories/        # Category management
└── README.md             # This file
```

## Quick Start

### Prerequisites

- [Bun](https://bun.sh/) (for frontend)
- [.NET 9 SDK](https://dotnet.microsoft.com/download/dotnet/9.0)
- [PostgreSQL](https://www.postgresql.org/download/)

### Development Setup

1. **Clone the repository**
   ```bash
   git clone <repository-url>
   cd nextnet-shop
   ```

2. **Setup Backend**
   ```bash
   cd net-backend
   dotnet restore
   dotnet run
   ```
   Backend will run on `http://localhost:8080`

3. **Setup Frontend**
   ```bash
   cd next-frontend
   bun install
   bun dev
   ```
   Frontend will run on `http://localhost:3000`

### Environment Configuration

Create `.env.local` files in both frontend and backend directories with appropriate environment variables.

## Development Commands

### Frontend (next-frontend/)
- `bun dev` - Start development server with Turbopack
- `bun build` - Build for production
- `bun start` - Start production server
- `bun lint` - Run ESLint

### Backend (net-backend/)
- `dotnet run` - Start development server
- `dotnet build` - Build the application
- `dotnet test` - Run tests (if available)

## API Documentation

The backend API includes Swagger documentation available at `http://localhost:8080/swagger` when running in development mode.

### Main API Endpoints

- `/api/products` - Product management
- `/api/categories` - Category management  
- `/api/users` - User authentication and management
- `/api/cart` - Shopping cart operations
- `/api/orders` - Order processing

## Contributing

Please read the `CLAUDE.md` file for development guidelines and best practices.

## Deployment

- **Frontend**: Deployed to Vercel with automatic deployments from Git
- **Backend**: Deployed to Fly.io with PostgreSQL database
- The backend may take time to load on first start due to Fly.io's pausing of inactive services

## License

This project is private and proprietary.
