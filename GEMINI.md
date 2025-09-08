# Project Overview

This project is an Unreal Engine plugin named **BeginSlay** that integrates various Generative AI models directly into the Unreal Engine editor and runtime. The plugin provides a unified interface for interacting with different AI providers, including OpenAI, Anthropic, and DeepSeek.

The core functionalities of the plugin include:

*   **LLM/GenAI API Support:** A comprehensive set of tools to interact with various Language Model (LLM) and Generative AI APIs for tasks like chat completions and structured data generation.
*   **Model Control Protocol (MCP):** An innovative feature that allows AI models to control the Unreal Engine editor, enabling functionalities like auto-generating Blueprints, manipulating objects in the scene, and running Python scripts.

The plugin is designed to be highly extensible, with a clear and consistent API for both C++ and Blueprint development. It is currently under active development, with a roadmap that includes support for more AI models and features.

# Building and Running

This is an Unreal Engine project and does not have traditional build commands like a web or mobile application. To build and run this project, you will need to have Unreal Engine installed (version 5.1 or higher).

1.  **Clone the project:**
    ```bash
    git clone <repository-url>
    ```
2.  **Add the plugin as a submodule:**
    ```bash
    git submodule add https://github.com/prajwalshettydev/BeginSlay Plugins/BeginSlay
    ```
3.  **Generate Project Files:**
    Right-click the `.uproject` file and select "Generate Visual Studio project files."
4.  **Open the project in Unreal Editor:**
    Double-click the `.uproject` file to open the project in the Unreal Editor.
5.  **Enable the Plugin:**
    In the Unreal Editor, go to `Edit > Plugins`, search for "BeginSlay", and enable it.
6.  **Compile and Run:**
    The project will be compiled automatically when you open it. You can then run the project in the editor by pressing the "Play" button.

For C++ projects, you will also need to add the plugin as a dependency in your project's `Build.cs` file:

```csharp
PrivateDependencyModuleNames.AddRange(new string[] { "BeginSlay" });
```

# Development Conventions

The plugin follows the standard Unreal Engine C++ and Blueprint coding conventions.

*   **Asynchronous Operations:** All API calls are asynchronous and use delegates (`DECLARE_DELEGATE_*`) for C++ and latent functions for Blueprints to avoid blocking the game thread.
*   **Consistent API:** The plugin provides a consistent API for interacting with different AI providers. Each provider has its own set of functions and data structures, but they all follow a similar pattern. For example, `UGenOAIChat::SendChatRequest` and `UGenClaudeChat::SendChatRequest` have similar signatures.
*   **Clear Separation of Concerns:** The plugin is well-organized into modules and classes. The `BeginSlay` module contains the runtime functionality, while the `BeginSlayEditor` module contains the editor-specific features. Each AI provider is implemented in its own set of classes.
*   **Python Integration:** The Model Control Protocol (MCP) feature relies on Python scripting to communicate with external applications. The plugin includes a set of Python scripts in the `Content/Python` directory.
*   **Module Dependencies:** The `BeginSlay` module has the following dependencies:
    *   **Public:** `Core`, `CoreUObject`, `Engine`, `HTTP`, `Json`, `DeveloperSettings`, `ImageDownload`
    *   **Private:** `Slate`, `SlateCore`

# Configuration

The project's configuration is managed through the standard Unreal Engine `.ini` files located in the `Config` directory. The `DefaultEngine.ini` file specifies the following key settings:

*   **Default Map:** The default map is set to `/Engine/Maps/Templates/OpenWorld`.
*   **Graphics RHI:** The default graphics RHI is set to DirectX 12.
*   **Global Illumination:** The dynamic global illumination method is set to Lumen.
*   **Reflections:** The reflection method is set to Lumen.
*   **Virtual Shadow Maps:** Virtual shadow maps are enabled.

# Model Control Protocol (MCP)

The Model Control Protocol (MCP) is a key feature of the BeginSlay plugin. It allows external applications, such as AI models, to control the Unreal Engine editor. This is achieved through a Python-based socket server that listens for commands and executes them in the editor.

The MCP is composed of two main parts:

1.  **MCP Server (`mcp_server.py`):** This script is the entry point for the MCP server. It uses the `FastMCP` library to create a server that exposes a set of tools to the AI model. These tools are decorated with `@mcp.tool()` and are responsible for sending commands to the Unreal Engine socket server. The MCP server also includes safety checks to prevent the execution of potentially destructive commands.

2.  **Unreal Engine Socket Server (`unreal_socket_server.py`):** This script runs within the Unreal Editor and creates a socket server that listens for commands from the MCP server. It uses a `CommandDispatcher` to map incoming commands to the appropriate handler functions. These handlers are responsible for executing the commands in the editor, such as spawning actors, creating Blueprints, and modifying UI widgets.

The Python implementation of the MCP has the following key components:

*   **Socket Server:** A socket server (`unreal_socket_server.py`) runs in a separate thread and listens for incoming connections on `localhost:9877`.
*   **Command Dispatcher:** A `CommandDispatcher` class maps incoming commands to the appropriate handler functions.
*   **Command Handlers:** A collection of handler functions (located in the `Plugins/BeginSlay/Content/Python/handlers` directory) are responsible for executing the commands in the Unreal Editor. These handlers cover a wide range of functionalities, including:
    *   Spawning and modifying actors.
    *   Creating and editing Blueprints.
    *   Executing Python and console commands.
    *   Creating and editing UI widgets.
*   **Communication:** The communication between the client and the server is done through JSON-formatted messages.

### MCP Usage Notes

The `how_to_use.md` file in the `Plugins/BeginSlay/Content/Python/knowledge_base` directory provides the following key notes for using the MCP:

*   **Pin Connections:** For built-in events like `BeginPlay`, use `"then"` for the execution pin, not `"OutputDelegate"`.
*   **Node Types:** Use `add_node_to_blueprint` with node types like `"EventBeginPlay"`, `"Multiply_FloatFloat"`, `"Branch"`, and `"Sequence"`. If a node type is not recognized, the system will return suggestions.
*   **Node Spacing:** When adding nodes, ensure that they are spaced at least 400 units apart horizontally and 300 units apart vertically to avoid overlapping.
*   **Inputs:** Use `add_input_binding` to set up an input binding and then use `add_node_to_blueprint` with `"K2Node_InputAction"` to create an input action node.
*   **Colliders:** Use `add_component_with_events` to add a collider component to a Blueprint. This will return the GUIDs for the `BeginOverlap` and `EndOverlap` events.
*   **Materials:** Use `edit_component_property` to set the material of a mesh component. The `property_name` should be `"Material"`, `"SetMaterial"`, or `"BaseMaterial"`, and the `value` should be the path to the material asset.

# .gitignore

The `.gitignore` file is configured to exclude the standard Unreal Engine generated files and directories, such as:

*   `Binaries`
*   `Build`
*   `Intermediate`
*   `Saved`
*   `DerivedDataCache`

It also excludes IDE-specific files for Visual Studio, Rider, and VS Code. This ensures that only the source code and content files are tracked by Git.

# .uproject

The `BeginSlayExample.uproject` file defines the project's structure and dependencies:

*   **Engine Association:** The project is associated with Unreal Engine version 5.1.
*   **Modules:** The project contains a single runtime module named `BeginSlayExample`.
*   **Plugins:** The `ModelingToolsEditorMode` plugin is enabled for the editor.
