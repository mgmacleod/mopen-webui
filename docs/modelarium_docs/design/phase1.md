# Modelarium Phase 1 Design

## Overview

Modelarium Phase 1 implements the core functionality for managing Ollama models and modelfiles. The system provides a web interface for creating and managing models through Ollama, with emphasis on reusable modelfile components.

## Technology Stack

- **Frontend**:
  - Angular with TypeScript
  - Angular Material for UI components
  - NgRx for state management

- **Backend**:
  - NestJS-based API server
  - PostgreSQL database

- **Infrastructure**:
  - Docker containers for all components (frontend, backend, database)
  - Docker Compose for local development
  - Multi-stage builds for production
  - Ollama running as separate service

## Core Features

### 1. Modelfile Components

#### Element Schema
- Individual modelfile elements stored as separate, reusable components:
  - Base models (FROM)
  - Templates
  - System messages
  - Parameter sets
- Each element includes:
  - Content
  - Source URL/attribution (optional)
  - Description/notes
  - Tags/categories
  - Creation/modification timestamps

#### Element Management
- CRUD operations for all modelfile elements
- Basic search and filtering
- Basic element validation

### 2. Modelfile Management

#### Modelfile Schema
- Complete modelfiles stored as compositions of elements
- Each modelfile includes:
  - References to its component elements
  - Name and description
  - Tags/categories
  - Creation/modification timestamps
  - Source URL/attribution (optional)

#### Modelfile Assembly
- Basic interface for combining elements into complete modelfiles
- Element selection and arrangement
- Generate modelfile text for Ollama API

### 3. Ollama Integration

#### API Support
- Create Model
- List Local Models
- Show Model Information
- Delete Model
- Show Modelfile Content

#### Configuration
- Support for remote Ollama instances
- Basic connection validation
- Configurable endpoint settings

### 4. User Interface

#### Model Management
- Basic dashboard for model overview
- Simple model creation interface
- Model information display
- Essential model actions (create, delete)

#### Modelfile Editor
- Basic element selection and assembly
- Simple parameter configuration
- Basic validation feedback

## Technical Considerations

### Database Design
- Normalized schema for modelfile elements and complete modelfiles
- Basic search indexing
- Efficient element-to-modelfile relationships

### State Management
- NgRx store design for:
  - Model list and status
  - Modelfile assembly state
  - API operation status
  - UI state (selected elements, form state)

### Docker Configuration
- Separate containers for:
  - Angular frontend
  - TypeScript backend
  - PostgreSQL database
- Development and production configurations
- Basic health checks
- Volume management for persistent data

### API Design
- RESTful endpoints for core operations
- Error handling and validation
- OpenAPI documentation
