# AI-Dev-Blueprint: SELF-DISCOVER Workflow

A streamlined, AI-assisted project workflow template using the research-backed SELF-DISCOVER framework with domain-specific add-ons.

## Overview

This repository provides a minimal, efficient framework for AI-assisted project development across various domains. It uses a single, unified workflow prompt that guides AI agents through a structured process of task planning, execution, and validation.

## Key Features

- **SELF-DISCOVER Framework**: Research-backed approach to AI reasoning and task execution
- **Modular Add-on System**: Domain-specific knowledge, reasoning modules, and templates
- **Structured Task Management**: Hierarchical task breakdown with rich metadata
- **Minimal Overhead**: Clean, streamlined approach with only essential files

## Core Files

- **workflow.md**: The master AI workflow prompt with SELF-DISCOVER framework
- **.agent_rules.md**: Behavioral rules for the AI agent and project standards
- **TODO.md**: Structured task management with metadata and hierarchical organization

## Add-on System

The workflow includes a modular add-on system that extends its capabilities for specific domains:

- **Qlik Add-on**: Specialized knowledge and templates for Qlik script development
- **Full-Stack Add-on**: Web application development using the bhvr stack (Bun, Hono, Vite, React)

Each add-on contains:
- Domain-specific knowledge
- Specialized reasoning modules
- Validation criteria
- Templates and examples

## How It Works

1. **Domain Detection**: The workflow automatically detects the domain of your task
2. **Add-on Loading**: Relevant domain-specific knowledge and templates are loaded
3. **Reasoning Structure**: A custom reasoning approach is created for your specific task
4. **Task Execution**: The task is completed following the structured plan
5. **Validation**: Results are verified against domain-specific criteria

## Getting Started

1. Clone this repository
2. Describe your task or project to the AI agent
3. The agent will automatically:
   - Create/update TODO.md with a structured task breakdown
   - Apply appropriate domain-specific knowledge
   - Execute tasks in priority order
   - Track and report progress

## Creating Custom Add-ons

To create a new domain-specific add-on:

1. Create a directory in `addons/` with your domain name
2. Add the following files:
   - `domain_knowledge.md`: Core concepts and best practices
   - `reasoning_modules.md`: Specialized reasoning approaches
   - `validation_criteria.md`: Success criteria and validation methods
   - `templates/`: Domain-specific templates and examples

## License

MIT
