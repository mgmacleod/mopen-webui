# Modelarium Overview

Modelarium is a managment system for large language models. The overall goal is to support model discovery, tracking, archival, and conversion (quantization) across a number of platforms, formats and inference backends. More specifically, we want to follow models from the time they are first posted in raw, unquantized form on Hugging Face (or similar platform), through the time that they become supported in quantization formats such as GGUF (llama.cpp) or Apple's MLX, through the time that they are supported and available for use in inferences engines like Ollama. The goal will be to gather, coordinate, and integrate information about what works and what doesn't with respect to the deployment and configuration of these models for specific use cases.

That said, this project is first and foremost a personal project with at most one guaranteed user. Therefore, we're going to start where I am:

_I use Ollama to run models in GGUF (and soon MLX) format for the purposes of code generation/analysis, tool calling/automation, and research/knowledge exploration._

The most immediate need is to get a handle on the best settings to use for a core set of model families (Llama, Mistral, Qwen, Granite) in general terms. Phase 1 is focused on basic Ollama API functionality with a focus on the Modelfile spec and using it as a way to represent composable subunits of configuration across models. (Ultimately, I'll want to track this, but for now, we're just getting started.)

### Phase 1: Basic Ollama API Functionality

#### Goals:

- Parse text modelfiles into component elements.
- Generate text modelfiles from component elements.
- View and extract elements from existing Ollama models.
- Copy/paste support for full modelfiles or individual elements.
- Preview parsed elements before saving to database.

#### Technical Considerations:

- **Database Design**: A normalized schema will be used for modelfile elements and complete modelfiles. Basic search indexing and efficient relationships will be implemented.
- **State Management**: NgRx store design will be utilized for managing model list, modelfile assembly state, API operation status, and UI state (selected elements, form state).
- **Docker Configuration**: Separate containers for Angular frontend, TypeScript backend, and PostgreSQL database. Development and production configurations will be set up with basic health checks and volume management.
- **API Design**: RESTful endpoints will be designed for core operations, error handling and validation will be implemented, and OpenAPI documentation will be created.

### Phase 2: Advanced Features

#### Goals:

- Enhance modelfile assembly features with visual interface enhancements, preview of complete modelfiles, advanced validation of element combinations, save/load draft modelfiles, export modelfile to text format.
- Integrate advanced Ollama API support for copy model, pull model, push model, search local models, and enhanced model information display. Implement additional features like connection status monitoring, connection pooling, failover support.
- Improve user interface with advanced model management features such as dashboard features, model creation wizard, advanced model action controls, batch operations, model comparison views. Enhance modelfile editor with visual editor enhancements, advanced parameter configuration, template preview, enhanced validation feedback, syntax highlighting, auto-completion.

#### Features:

- **Modelfile Assembly**:
  - Visual interface enhancements for better user experience.
  - Preview of complete modelfile to ensure accuracy before saving.
  - Advanced validation of element combinations to prevent errors.
  - Save/load draft modelfiles for easy editing and revisiting.
  - Export modelfile to text format for easier sharing or integration with other systems.
  - Copy/paste support for full modelfiles or individual elements.
- **Ollama Integration**:
  - Additional API support for copy model, pull model, push model, search local models, and enhanced model information display.
  - Enhanced health checks to monitor the connection status of Ollama models.
  - Connection status monitoring to ensure continuous connectivity.
  - Connection pooling to optimize resource usage and improve performance.
  - Failover support to handle network issues or server downtime gracefully.
- **User Interface**:
  - Advanced model management features such as dashboard features, model creation wizard, advanced model action controls, batch operations, and model comparison views.
  - Enhance modelfile editor with visual editor enhancements, advanced parameter configuration, template preview, enhanced validation feedback, syntax highlighting, and auto-completion.
