# Open WebUI - Model Lab Fork

This project is a fork of Open WebUI (the popular frontend for working with local LLMs) with a focus on model management, experimentation, and tracking. In particular, this fork will be focus on a more direct integration of the Ollama inference engine.

## Rationale

Open WebUI supports some level of similar functionality already. The main issue is that the model configurations are custom and internal to Open WebUI, and so are only available in the app itself or via its API. That's fine and all, except that lots of things already support the Ollama API directly (whereas virtually nothing else supports the Open WebUI API), so I'd rather centralize the management of models in Ollama. Indeed, Ollama's strengths in this area are part of why I chose it as my preferred inference backend.

In order to maintain backward compatibility, I don't want to modify the existing model management functionality; rather, I want to add a new 'section' to the app (both frontend and backend) that will encapsulate the new functionality and have a minimal impact on the surrounding code.

No decision has been made about whether to try to get these changes merged back upstream, but I want to keep the option open.

## Current goals

1. Add the ability view the [modelfile](https://github.com/ollama/ollama/blob/main/docs/modelfile.md) of an existing Ollama model, make modifications, and save the results via the Ollama API
   1. Common examples:
      1. Correcting a minor issue with the prompt template of a model (or just tweaking it to improve performance)
      2. Adding custom system prompts
      3. Adjusting parameters to optimal settings
   2. This should be implemented as a new 'Lab' section of the app to keep it separate from existing functionality
2. Expand the database schema to model modelfiles at a more granular level and include the various components of a modelfile (such as FROM, TEMPLATE, and SYSTEM elements as well as collections of PARAMETER and MESSAGE elements)
   1. The goal is to build up a library of 'presets' that can be used as building blocks for configuring models
      1. Models often share prompt templates
      2. System prompts can be applied across models
   2. Make this expanded representation available from the new Lab section of the UI
