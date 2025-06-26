### Project Overview

**Gemini Repomix Assistant** is a sophisticated, full-stack AI-powered coding assistant designed to help developers interact with, understand, and modify entire code repositories. It's not just a simple chatbot; it's a complete integrated development environment that bridges the gap between natural language instructions and hands-on coding.

The core concept is to use the **Repomix** tool to create a comprehensive, single-file representation of a Git repository. This "packed" file is then used as a massive context for a large language model (like Google Gemini or OpenAI's GPT series). The user can then chat with the AI about the codebase, ask for modifications, generate new code, and then review and deploy these changes directly to their local file system or even back into their running VS Code workspace.

The application is architected as a three-part system:
1.  A **web-based user interface** where all user interaction occurs.
2.  A **Node.js/Express backend** that orchestrates API calls, data storage, and business logic.
3.  A **VS Code extension** that provides a real-time bridge between the web UI and the user's local development environment.

### Core Functionality

1.  **Repository Analysis**: Users can provide a URL to a Git repository. The backend uses the `repomix` CLI tool to fetch the repository, analyze its structure, and pack the contents of relevant files into a single, context-rich file.
2.  **AI-Powered Chat**: The primary interface is a chat window. The packed repository file is used as context, allowing the user to ask complex questions, request refactoring, or generate new code with full awareness of the entire project's structure and contents.
3.  **Multi-Model Support**: While named for Gemini, the application is designed to support multiple AI providers. It has built-in services for both Google Gemini and OpenAI models, with a configuration system that allows users to switch between them.
4.  **Code Generation and Modification**: The AI's responses often include code blocks. The application intelligently parses these blocks, identifying the target file paths from comments within the code.
5.  **Interactive Diff and Comparison**: Before applying any changes, the user can view a "diff" of the AI's suggested modifications against the original file content. This provides a clear, color-coded comparison, essential for verification.
6.  **Hybrid Deployment System**: This is a standout feature. Users can choose to deploy the AI-generated code changes to two targets:
    *   **Local Folder**: Using the browser's File System Access API, the app can directly read, write, and create backups in a user-selected local directory.
    *   **VS Code**: Through a real-time connection, the app can send commands to the VS Code extension, which then applies the file changes directly within the user's open workspace.
7.  **Workspace Integration & Automation**: The VS Code extension not only applies file changes but can also run user-configured build commands (e.g., `npm install && npm run build`) in the integrated terminal after a deployment, creating a powerful automation workflow.

### System Architecture

The project is composed of three distinct but interconnected components:

#### 1. Backend (`backend/`)
*   **Technology**: Node.js, Express.js, TypeScript.
*   **Responsibilities**:
    *   **API Server**: Exposes a REST API for all frontend operations.
    *   **User Authentication**: Middleware (`authMiddleware.ts`) validates JWTs from Supabase on every protected request.
    *   **Repomix Orchestration**: The `repomixRoutes.ts` file handles requests to run the `repomix` CLI tool, managing the process and storing the output files.
    *   **AI Service Proxy**: The `geminiRoutes.ts` acts as a secure proxy to the AI model providers. It retrieves the user's stored API key (from Supabase), constructs the prompt with the repository context, and forwards the request to either the `geminiService.ts` or `openaiService.ts`. This prevents exposing user API keys to the client.
    *   **Database Operations**: Interacts with the Supabase database to manage user configurations, API keys, saved chat sessions, and deployment records.
    *   **Cost Management**: The `costService.ts` calculates the cost of each AI call based on token usage and stores it in the database.
    *   **Tool Integration**: The `mcpService.ts` (Model-Contex-Protocol service) and `googleWorkspaceService.ts` provide a framework for the AI to use external tools, such as searching Google Drive or reading a Gmail message.

#### 2. Frontend (`gemini-repomix-ui/`)
*   **Technology**: React, Vite, TypeScript, Redux Toolkit, TanStack Query, Tailwind CSS.
*   **Responsibilities**:
    *   **User Interface**: Provides the entire visual experience, including the authentication form, main chat workspace, file tree, code viewers, and configuration panels.
    *   **State Management**: It uses a hybrid approach:
        *   **Redux Toolkit**: For managing global UI state (like which panels are visible) and synchronous client-side state.
        *   **TanStack Query**: For managing all server state, including fetching repository lists, user configurations, chat history, and cost history. This handles caching, refetching, and mutations elegantly.
    *   **Component-Based UI**: Built with reusable React components for different UI sections like `ChatInterface`, `FileTree`, `ComparisonView`, `RepomixForm`, etc.
    *   **Context API**: Uses React's Context API (`WorkspaceContext`, `DeploymentContext`, `RealtimeContext`) to manage complex, cross-cutting concerns like the current workspace data, deployment logic, and the real-time connection status.
    *   **API Client**: A robust `apiClient.ts` handles all communication with the backend, automatically attaching authentication tokens.
    *   **Streaming Responses**: User has the ability to recieve responses as they are generated and prompts can be stopped.
    *   **Multimodal Support**: Images can be attached to prompts like screenshots of debug output.
    *   **Smart Tree View**: The user can select/deselect individual files as well as whole folders for a more focused analysis.
    *   **Syntax Highlighting**: Codeblocks are visualized with modern syntax highlighting.

#### 3. VS Code Extension (`repochatextension/`)
*   **Technology**: TypeScript, VS Code Extension API.
*   **Responsibilities**:
    *   **Real-time Communication**: Connects to a dedicated Supabase Realtime channel for the logged-in user. It listens for commands broadcasted by the web UI.
    *   **Presence Reporting**: Periodically reports its "online" status and the current workspace path back to the web UI, so the user knows the extension is active and ready.
    *   **File System Operations**: The `handlers.ts` file contains the logic to execute commands received from the UI. This includes:
        *   Creating, writing, or overwriting files in the user's workspace (`handleDeployFileRequest`).
        *   Undoing all uncommitted changes via `git checkout -- .` (`handleUndoAllRequest`).
        *   Running the `repomix` command on the local workspace (`handleGenerateFromVscodeRequest`).
    *   **Terminal Integration**: Can receive and execute shell commands (like build scripts) in a new VS Code terminal instance (`handleBuildRequest`).
    *   **Authentication**: Securely stores the user's Supabase session in the VS Code SecretStorage.

### Data Flow & Key Features in Detail

*   **Authentication**: Uses Supabase Auth. A user can sign up/in via email/password or GitHub OAuth. The session JWT is stored and used to authenticate all backend API calls. The VS Code extension also uses this session to authenticate with the real-time service.
*   **Real-time Bridge**: The UI and the VS Code extension communicate via a **Supabase Realtime channel** specific to the user (`user-comms:<user-id>`). This is a publish/subscribe system that allows for instant, bidirectional communication without a direct network connection between the two, which is crucial for a web application to interact with a local desktop application.
*   **Cost Tracking**: The backend meticulously calculates the cost of every AI call using pricing data stored in `gemini_models_config.json`. The frontend presents this data in a detailed `CostHistoryModal`, complete with daily charts and model-based breakdowns.
*   **Customization**: Users have a high degree of control. They can:
    *   Switch between different Gemini and OpenAI models.
    *   Tune model parameters like temperature and top-p (`ModelTuningPanel.tsx`).
    *   Create and manage a library of custom system prompts to tailor the AI's behavior (`SystemPromptEditor.tsx`).
    *   Enable or disable AI tool usage.
*   **Undo/Recovery**: The local deployment feature includes a robust backup system. Before overwriting a file, a timestamped backup is created in a hidden `.repomix_backups` directory. The `useUndoManager` hook orchestrates restoring these backups. The VS Code deployment relies on Git for recovery (`git checkout`).

In summary, this project is a powerful and well-architected developer tool that cleverly combines a web application, a backend service, and a local editor extension to create a seamless AI-assisted coding experience. It demonstrates a deep understanding of modern web development practices, state management, and system design.
