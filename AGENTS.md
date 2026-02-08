# Agents for Blender MCP

This document contains guidelines and rules for agents working with Blender through the Model Context Protocol (MCP).

## Overview

This repository provides context and rules for AI agents to assist with Blender-related tasks, including 3D modeling, animation, scripting, and workflow automation.

## Agent Guidelines

### Blender Knowledge Requirements

Agents should be familiar with:
- Blender's Python API (bpy)
- 3D modeling concepts and workflows
- Blender's node systems (Geometry Nodes, Shader Nodes, Compositor Nodes)
- Animation and rigging workflows
- Blender's UI and interaction patterns

### Best Practices

1. **Script Safety**: Always validate user input and handle errors gracefully in Blender Python scripts
2. **Version Compatibility**: Be aware of Blender version differences when providing solutions
3. **Performance**: Consider performance implications of operations, especially with large datasets
4. **Documentation**: Reference official Blender documentation when applicable
5. **Context Awareness**: Understand the current state of the Blender project before making suggestions

### Common Tasks

- Creating and manipulating 3D objects
- Setting up materials and shaders
- Animating objects and properties
- Generating procedural geometry with Geometry Nodes
- Writing and debugging Python scripts for Blender
- Automating repetitive workflows

## Usage

This file serves as context for agents to better understand Blender-specific requirements and provide more accurate, helpful assistance to users working with Blender.
