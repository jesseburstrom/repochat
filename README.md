This document provides a detailed description of the Gemini Repomix Assistant, a comprehensive system designed for AI-powered code analysis and interaction. The system is composed of three main parts: a backend service, a web-based user interface, and a Visual Studio Code extension.

### System Architecture Overview

The system is a full-stack application with a project-centric and collaborative architecture.

*   **Backend (`backend/`)**: A Node.js application built with Express and TypeScript. It serves as the central hub, handling API requests, business logic, authentication, and communication with external services like AI models and databases.
*   **Frontend (`gemini-repomix-ui/`)**: A modern single-page application (SPA) built with React, Vite, and TypeScript. It provides a rich, interactive user interface for managing projects, chatting with the AI, and viewing code.
*   **VS Code Extension (`repochatextension/`)**: An extension for Visual Studio Code that integrates the user's local development environment directly with the web application, enabling features like local code analysis, file deployment, and running build scripts.

Communication between the frontend and the VS Code extension is facilitated by a real-time messaging layer using Supabase Realtime, allowing for seamless interaction between the web UI and the local IDE.

---

### Backend Details

The backend is responsible for the core logic of the application.

**Key Technologies:**
*   **Runtime/Framework:** Node.js, Express.js
*   **Language:** TypeScript
*   **Core Dependencies:** `@google/generative-ai`, `openai`, `@supabase/supabase-js`, `googleapis`.

**Core Features & Services:**

1.  **Authentication & Authorization:**
    *   Integrates with Supabase for user management, supporting email/password and GitHub OAuth.
    *   A robust `authMiddleware.ts` protects API endpoints using JWT Bearer tokens. It also supports token authentication via query parameters for specific use cases like browser-based OAuth redirects.

2.  **AI Model Routing & Interaction (`services/geminiService.ts`, `openaiService.ts`, `openrouterService.ts`):**
    *   The backend acts as a versatile proxy and router for multiple AI providers. Based on the model selected by the user (`modelCallName`), incoming chat requests are directed to the appropriate service.
    *   It supports Gemini, OpenAI, and a wide array of models via OpenRouter. The available models are dynamically configured through `gemini_models_config.json`, which includes pricing, capabilities, and API identifiers.
    *   Handles both standard request-response and real-time streaming chat using Server-Sent Events (SSE).

3.  **Tool Use & Function Calling (MCP - Model-facing Capability Platform):**
    *   A key architectural feature is the MCP, defined in `services/mcpService.ts` and configured via `mcp_config.json`.
    *   This service abstracts tool definitions (e.g., searching Google Drive, performing a Google search) and presents them to the AI models in their required format (Gemini `functionDeclarations` or OpenAI `tools`).
    *   When an AI model decides to use a tool, the MCP dispatches the request to the appropriate internal handler. For example, a request for `search_drive_files` is routed to the `googleWorkspaceService.ts`. This makes the system highly extensible for adding new tools.

4.  **Google Workspace Integration (`services/googleWorkspaceService.ts`):**
    *   Provides a rich set of tools for interacting with a user's Google account.
    *   Handles the entire OAuth2 flow for authorization, securely storing tokens in the Supabase database.
    *   Implements functions for Drive (search, read, create), Calendar (list, get events), Gmail (search, read, send), Sheets, and Chat.

5.  **Project & Session Management (`services/projectService.ts`, `chatSessionService.ts`):**
    *   The application is built around a project-centric model. Users can create projects, add repositories, and invite other users with specific roles (`admin`, `member`, `viewer`).
    *   Chat sessions are associated with either a user's personal space or a specific project, allowing for collaborative chat histories within a team context.

6.  **`Repomix` Integration (`routes/repomixRoutes.ts`):**
    *   The backend executes the `repomix` Command-Line Interface (CLI) as a child process to analyze remote Git repositories.
    *   It handles user-defined include/exclude patterns and stores the generated output files on the server.
    *   Metadata about each generated file (user, repo URL, file size, token count) is saved to the database.

7.  **User Configuration (`services/userApiKeyService.ts`):**
    *   Manages all user-specific settings, such as API keys for various AI providers, the last selected model, model tuning parameters (temperature, topP, etc.), and custom system prompts.
    *   Includes an endpoint (`/api/user-config/users`) to list all registered users, facilitating the "add member" feature in projects.

8.  **Cost Tracking (`services/costService.ts`):**
    *   After every AI API call, this service is invoked to record detailed usage statistics, including the model used, input/output token counts, and the calculated cost based on the pricing information in `gemini_models_config.json`. This data is stored per-user in the database.

---

### Frontend UI Details

The frontend provides a responsive and feature-rich interface for users.

**Key Technologies:**
*   **Framework/Tooling:** React, Vite, TypeScript, Tailwind CSS
*   **State Management:** Redux Toolkit (for global UI state) and TanStack Query (for server state management).
*   **Syntax Highlighting:** Shikiji

**Core Features & Components:**

1.  **State Management Strategy:**
    *   **TanStack Query** is the primary driver for data fetching, caching, and mutations, providing a robust and efficient way to interact with the backend. This is seen in the various custom hooks like `useProjects`, `useChatSessions`, and `useUserConfiguration`.
    *   **Redux Toolkit** is used for managing purely client-side global state, such as UI toggles (`uiSlice`), authentication status (`authSlice`), and transient messages (`statusSlice`).

2.  **Workspace & Layout:**
    *   The `MainWorkspaceLayout.tsx` uses `react-resizable-panels` to create a professional, multi-pane UI that can be toggled between a standard view and a "fullscreen" developer-focused view.
    *   **WorkspaceContext** is a central provider that manages the state of the currently loaded repository, including its file tree, file content, and user selections.

3.  **Code Interaction:**
    *   **`FileTree.tsx`**: Displays the directory structure of a loaded repository, allowing users to select one or more files to be included as context in their prompts.
    *   **`CodeBlockRenderer.tsx`**: A sophisticated component that renders code blocks with syntax highlighting. It automatically detects file paths commented in the code and provides "Copy", "Locate" (to edit the path), and "Compare" functionality.
    *   **`ComparisonView.tsx`**: When a user clicks "Compare," this component shows a side-by-side diff of the original file content against the AI's suggested changes.

4.  **Deployment & Actions (`DeploymentPanel.tsx`, `useCodeDeployer.ts`):**
    *   Users can apply AI-generated code changes directly to their local filesystem.
    *   The **Deployment Target** can be switched between "Local Folder" (using the browser's File System Access API) and "VSCode" (via the extension).
    *   The system supports creating backups before overwriting local files and provides an "Undo All" feature.
    *   It also integrates a **Build Script Manager**, allowing users to define, save, and run shell scripts within their VS Code workspace directly from the UI.

5.  **Project Management (`ProjectManagerPanel.tsx`, `useProjects.ts`):**
    *   A complete interface for creating projects, managing repositories associated with them, and inviting/managing project members and their roles.

---

### VS Code Extension Details

The extension acts as a vital bridge between the web UI and the developer's local machine.

**Key Functionality:**

1.  **Real-time Connection (`realtimeService.ts`):**
    *   Connects to a user-specific Supabase Realtime channel to listen for commands from the web UI.
    *   Uses a **presence** system to notify the UI that the extension is online and active.
    *   Includes a `ConnectionManager` that implements a health check to automatically re-establish the connection if it drops.

2.  **Local Workspace Integration (`commands/`):**
    *   **Generate from Workspace**: On request from the UI, it runs `repomix` on the current open folder and streams the result back to the UI.
    *   **File Deployment**: Listens for `request_deploy_file` events and uses the `vscode.workspace.fs` API to write or create files in the user's workspace.
    *   **Undo Changes**: Listens for `request_undo_all_deployments_vscode` and runs `git checkout -- .` to revert uncommitted changes.
    *   **Run Scripts**: Listens for requests to run build scripts, executes them in the local terminal, and streams the output back to the UI's status log.
    *   **Run Diagnostics (Linting)**: On request, it can open files in the editor, wait for a configurable delay, and then collect all diagnostic information (errors/warnings) from VS Code's language servers, sending the results back to the UI for analysis by the AI.

3.  **Local Project Scaffolding (`commands/projectSetupHandlers.ts`):**
    *   Provides a powerful feature to create new projects from predefined templates directly from the UI.
    *   When a user initiates project creation, the extension prompts them to select a parent directory. It then creates the new project folder and opens it.
    *   An autorun task then takes over, executing a series of steps defined in the template: running base commands (like `npm create vite`), installing dependencies, creating configuration files (like `tailwind.config.js`), and finally, running `repomix` on the newly created project to load it into the UI.

4.  **Multi-User Authentication (`auth.ts`):**
    *   Securely stores session tokens for multiple user accounts using VS Code's `SecretStorage`.
    *   Provides a webview in the sidebar for users to log in, switch between saved accounts, or log out.
