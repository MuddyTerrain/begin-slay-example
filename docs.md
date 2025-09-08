# BeginSlay: Technical Architecture & Implementation Plan

This document provides a detailed technical specification for the BeginSlay Unreal Engine plugin, its tiered feature set, the backend infrastructure, and the go-to-market strategy.

## 1. Product Vision & Feature Tiers

### 1.1. BeginSlay Standard
- **Distribution:** One-time purchase, Fab.com marketplace.
- **Technical Scope:** A self-contained Unreal Engine Editor Module (`BeginSlay`) with no external server dependencies.
- **Core Features:**
    - **Model Control Protocol (MCP):** The foundational Python-based socket server (`unreal_socket_server.py`) and command handlers for direct editor manipulation.
    - **Core LLM Integrations:** C++ classes for interfacing with standard chat completion APIs (OpenAI, Anthropic, etc.).
    - **UI:** The basic plugin settings panel and editor window.
    - **Upgrade Path:** UI elements that explicitly link to the Pro version's sales page.

### 1.2. BeginSlay Pro
- **Distribution:** Monthly subscription ($8/month), Gumroad via `store.muddyterrain.com`.
- **Technical Scope:** A second Editor Module (`BeginSlayPro`) that has a direct dependency on the `BeginSlay` module. This module contains features that require external server communication for licensing and advanced AI processing.
- **Core Features:**
    - All Standard features.
    - **RAG-Powered Blueprint IntelliSense:** The primary value proposition. A sophisticated system for context-aware Blueprint node generation from natural language prompts.
    - **Advanced Model Support:** Integration with premium or specialized AI models.
    - **License Verification System:** A robust, server-validated mechanism to ensure only active subscribers can access Pro features.

## 2. Core Technical Architecture

### 2.1. Unreal Engine Plugin Module Architecture
The project is architected as a single plugin containing two distinct modules to enforce a clean separation between the Standard and Pro feature sets.

- **`BeginSlay` Module:**
    - **Type:** Editor Module.
    - **Responsibilities:** Contains all core, self-contained logic. This includes the C++ subsystems, the MCP Python backend, and the base UI panels. It is the foundational layer.
- **`BeginSlayPro` Module:**
    - **Type:** Editor Module.
    - **Dependencies:** Explicitly depends on the `BeginSlay` module in its `.uplugin` descriptor and `BeginSlayPro.Build.cs` file. This allows Pro features to directly call and extend Standard features.
    - **Responsibilities:** Houses all functionality requiring a subscription. This includes the RAG Manager, the Blueprint Generator UI, and the license verification handshake logic.

This two-module approach is technically superior because it allows for a single codebase while enabling separate packaging and distribution. The Standard version can be shipped without any of the Pro code, and vice-versa.

### 2.2. RAG-Powered IntelliSense System (Pro Feature)

This system is composed of two primary sub-systems: **Indexing** and **Retrieval/Generation**.

#### 2.2.1. The Indexing Sub-system
The goal is to create a comprehensive, structured database of every possible action a user can take within the Blueprint editor. This is achieved by the `FBeginSlayRAGManager` class.

- **Mechanism:**
    1.  On editor startup, an asynchronous task (`FAsyncTask<FRAGScanTask>`) is launched to prevent UI freezing.
    2.  The task executes `ScanForBlueprintNodes`, which uses `TObjectIterator<UClass>` to get a reference to every `UClass` currently loaded in the editor's memory space.
    3.  For each `UClass`, it uses `TFieldIterator<UFunction>` to iterate over all its member functions.
    4.  A filter is applied to keep only functions marked with `FUNC_BlueprintCallable` or `FUNC_BlueprintPure`, and without the `DeprecatedFunction` metadata tag.
    5.  For each valid function, the system extracts a rich set of data into an `FBlueprintNodeInfo` struct:
        - **`FunctionName`**: `Function->GetName()`
        - **`NodeClassName`**: `Class->GetName()`
        - **`Category`**: `Function->GetMetaData(TEXT("Category"))`
        - **`Tooltip`**: `Function->GetMetaData(TEXT("Tooltip"))`
        - **Pins**: It performs a nested iteration with `TFieldIterator<FProperty>` on the function's properties. Any property (`FProperty*`) flagged as `CPF_Parm` is a pin. Its name, C++ type, and direction (input vs. output/return) are stored in an `FBlueprintPinInfo` struct.

- **Data Storage:**
    - The resulting `TArray<FBlueprintNodeInfo>` is stored in-memory within the `FBeginSlayRAGManager` singleton for the duration of the editor session. This provides the lowest possible latency for retrieval.
    - The `ExportToJSON` function is a debug utility to serialize this in-memory database to disk, it is not used for live operation.

#### 2.2.2. The Retrieval & Generation Pipeline
This is a multi-stage process designed to maximize accuracy while managing cost and latency.

1.  **Step 1: Intent Distillation (Fast LLM)**
    - **Input:** Raw user prompt (e.g., "make the character jump when I press the spacebar").
    - **Process:** The prompt is sent to a fast, low-cost LLM (e.g., GPT-3.5-Turbo). The system prompt instructs the model to act as a "tool identifier" and return a JSON array of relevant keywords and potential node names.
    - **Output:** `["InputAction", "EnhancedInput", "Jump", "Character"]`

2.  **Step 2: RAG Retrieval (Local)**
    - **Input:** The JSON array from Step 1.
    - **Process:** The `FBeginSlayRAGManager::FindRelevantNodes` function iterates through the in-memory database. The current implementation uses a simple keyword-matching algorithm with a scoring system to find the most relevant `FBlueprintNodeInfo` structs.
    - **Output:** A `TArray<FBlueprintNodeInfo>` containing the top 10-50 matching nodes, with all their pin and metadata information.

3.  **Step 3: Contextual Prompt Engineering**
    - **Input:** The original user prompt and the retrieved `TArray<FBlueprintNodeInfo>`.
    - **Process:** A new, highly detailed prompt is constructed for a powerful reasoning model.
        - The retrieved `TArray<FBlueprintNodeInfo>` is serialized into a clean JSON string.
        - A system prompt is created that defines the model's role, provides the retrieved node data as "context", and includes several examples of the T3D text format for copy-pasting nodes (few-shot prompting).
    - **Output:** A single, large prompt string sent to the powerful LLM.

4.  **Step 4: Generation & Editor Injection (Powerful LLM)**
    - **Input:** The context-rich prompt from Step 3.
    - **Process:** The prompt is sent to a state-of-the-art model (e.g., GPT-4). The model uses its reasoning capabilities to synthesize the user's request with the provided node data and T3D examples.
    - **Output:** A string of pure T3D text. This text is then passed to the Unreal Engine function `FEdGraphUtilities::ImportNodesFromText` to be pasted directly into the user's active `UEdGraph`.

#### 2.2.3. Roadmap & Technical Enhancements
- **Semantic Search:** The current keyword-based retrieval will be upgraded. This involves generating vector embeddings for each `FBlueprintNodeInfo` during indexing and storing them. User queries will also be embedded, and retrieval will be done via cosine similarity search, allowing the system to understand user *intent* (e.g., "create an enemy" would match the `Spawn Actor From Class` node).
- **Blueprint Asset Parsing:** The system will be extended to parse `.uasset` files. It will use the `AssetRegistry` module to find all `UBlueprint` assets, load them, and iterate through their `UbergraphPages`, `FunctionGraphs`, and `MacroGraphs` to index user-defined functions and logic, making the RAG system aware of the project's own codebase.

### 2.3. Licensing & Authentication Backend

- **Infrastructure:** A single `e2-micro` Google Compute Engine instance running Ubuntu. This is well within the GCP free tier and sufficient for the expected load.
- **API Server:** A Python server using the **FastAPI** framework.
- **Database:** A single **SQLite** database file. This is a zero-cost, serverless solution that is perfectly adequate for storing and managing license keys. The database schema will be a single table: `licenses (key TEXT PRIMARY KEY, status TEXT, gumroad_sale_id TEXT, last_checked_utc INTEGER)`.
- **Security:** The server will be placed behind a GCP Load Balancer configured for HTTPS, which will handle SSL termination. All sensitive credentials (Gumroad API key, JWT secret) will be stored as environment variables on the VM, not in the source code.

#### 2.3.1. Detailed Activation Handshake
1.  **Plugin -> Server:** The `BeginSlayPro` module makes a `POST` request to `https://store.muddyterrain.com/verify-license` with a JSON body: `{"license_key": "..."}`.
2.  **Server -> Gumroad:** The server receives the key and makes a server-to-server `POST` request to `https://api.gumroad.com/v2/licenses/verify` with the license key.
3.  **Server Logic:** If Gumroad's API confirms the license is valid and active, the server generates a **JSON Web Token (JWT)**.
    - **JWT Payload:** `{"sub": "gumroad_sale_id", "exp": "now + 7 days", "tier": "pro"}`
    - The JWT is signed using a strong secret key (e.g., a 256-bit key stored as an environment variable).
4.  **Server -> Plugin:** The server responds with the signed JWT.
5.  **Plugin Logic:** The plugin verifies the JWT's signature. If valid, it saves the JWT to a local config file within the project's `Saved/Config` directory and unlocks all Pro features. The plugin will schedule a silent re-validation task to run before the 7-day expiry.

#### 2.3.2. Gumroad Webhooks
- The server will expose a `POST /gumroad-webhook` endpoint.
- In Gumroad, this endpoint will be configured as the "Ping" URL.
- The server will handle `sale`, `cancellation`, and `refund` events to update the `status` field in the SQLite database in real-time, ensuring that access is revoked promptly if a subscription ends.