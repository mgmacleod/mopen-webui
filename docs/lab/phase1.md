# Open WebUI Lab fork - Phase 1 - Basic Ollama Model Editing

The goal of phase one is to implement basic Ollama model editing functionality. This phase should not involve any database or backend changes, only new UI elements.

## Overview

We will add a new section called 'Lab' that will be placed on the left sidebar under the existing 'Workspace' section. The Lab section will be based on a similar model to how the Workspace section works, with tabbed sub-pages for different functions (in the case of Workspace, these are 'Models', 'Knowledge', 'Prompts', 'Tools').

The first sub-page will be called 'Ollama' and have a dual panel layout that will allow for simultaneous viewing of existing models and creation of new models. The workflow would be something like this:

- look up a model to use as a reference on the left-hand panel and load its modelfile (probably also displaying some other details for reference, such as architecture, context length, capabilities, etc.)
- copy and paste elements from the reference model to the new one on the right-hand panel
- edit and tweak until satisfied
- save ('create') the model using the Ollama API

## Requirements

- Continue the established patterns of the Open WebUI project
  - Code organization and style
  - UI/UX conventions
- Use existing functionality where possible
  - All the Ollama functionality we need should already exist in the backend
- (Optional) Restrict the Lab section to admin users only using the existing auth mechanisms
