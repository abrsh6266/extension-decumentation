# Code Visualizer Extension - Detailed Explanation

## 📋 Table of Contents
1. [Project Overview](#project-overview)
2. [Architecture Overview](#architecture-overview)
3. [Step-by-Step Flow](#step-by-step-flow)
4. [Component Breakdown](#component-breakdown)
5. [Data Flow](#data-flow)
6. [Key Concepts](#key-concepts)

---

## 🎯 Project Overview

**What does this extension do?**
This VS Code extension visualizes GitHub repository structures as interactive graphs. It:
- Clones GitHub repositories
- Parses Python code to extract classes and functions
- Creates a visual graph showing the repository structure
- Provides multiple visualization layouts (tree, radial, force-directed, etc.)
- Allows you to click nodes to view code and navigate to definitions

**Why is it useful?**
- Understand large codebases quickly
- See relationships between files, classes, and functions
- Navigate code visually instead of browsing folders
- Get a bird's-eye view of project architecture

---

## 🏗️ Architecture Overview

The extension follows a **client-server architecture** within VS Code:

```
┌─────────────────────────────────────────────────────────┐
│                    VS Code Extension                     │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Extension Host (TypeScript/Node.js)             │   │
│  │  - extension.ts (Entry Point)                   │   │
│  │  - visualizationPanel.ts (Webview Provider)     │   │
│  │  - repoParser.ts (Code Parser)                  │   │
│  │  - visualizations/* (Layout Configs)            │   │
│  └──────────────────────────────────────────────────┘   │
│                          ↕ Messages                      │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Webview (HTML/CSS/JavaScript)                   │   │
│  │  - main.js (Cytoscape.js Graph)                  │   │
│  │  - main.css (Styling)                            │   │
│  │  - cytoscape.min.js (Graph Library)              │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

**Key Technologies:**
- **TypeScript**: Extension backend code
- **Cytoscape.js**: Graph visualization library
- **simple-git**: Git operations (clone, pull)
- **VS Code API**: Extension host communication

---

## 🔄 Step-by-Step Flow

### Step 1: Extension Activation

**File:** `src/extension.ts`

When VS Code loads the extension:

```6:27:src/extension.ts
export function activate(context: vscode.ExtensionContext) {
    console.log('Code Visualizer extension is now active');

    // Create the visualization provider
    visualizationProvider = new VisualizationViewProvider(
        context.extensionUri,
        context
    );

    // Register the webview view provider for the activity bar
    context.subscriptions.push(
        vscode.window.registerWebviewViewProvider(
            VisualizationViewProvider.viewType,
            visualizationProvider,
            {
                webviewOptions: {
                    retainContextWhenHidden: true
                }
            }
        )
    );
```

**What happens:**
1. VS Code calls `activate()` when the extension is first used
2. Creates a `VisualizationViewProvider` instance
3. Registers it as a webview provider (creates the UI panel)
4. Registers commands (cloneRepo, refresh, etc.)

**Why `retainContextWhenHidden: true`?**
- Keeps the webview alive when you switch tabs
- Prevents losing graph state when navigating away

---

### Step 2: Webview Creation

**File:** `src/visualizationPanel.ts`

When you open the visualization view:

```26:59:src/visualizationPanel.ts
    public resolveWebviewView(
        webviewView: vscode.WebviewView,
        context: vscode.WebviewViewResolveContext,
        _token: vscode.CancellationToken
    ) {
        this._view = webviewView;

        webviewView.webview.options = {
            enableScripts: true,
            localResourceRoots: [
                vscode.Uri.joinPath(this._extensionUri, 'media')
            ]
        };

        webviewView.webview.html = this._getHtmlForWebview(webviewView.webview);

        // Handle messages from webview
        webviewView.webview.onDidReceiveMessage(async message => {
            await this._handleMessage(message);
        });

        webviewView.onDidDispose(() => {
            this._disposeWatcher();
        });

        webviewView.onDidChangeVisibility(() => {
            if (webviewView.visible && this._repoData) {
                this._sendMessage({
                    command: 'updateGraph',
                    data: this._repoData
                });
            }
        });
    }
```

**What happens:**
1. VS Code calls `resolveWebviewView()` when the panel is opened
2. Sets up webview options (allows scripts, defines resource paths)
3. Generates HTML content (includes CSS, JavaScript files)
4. Sets up message listener (for communication between webview and extension)
5. Sets up lifecycle handlers (dispose, visibility changes)

**The HTML Structure:**
```354:457:src/visualizationPanel.ts
    private _getHtmlForWebview(webview: vscode.Webview): string {
        const scriptUri = webview.asWebviewUri(
            vscode.Uri.joinPath(this._extensionUri, 'media', 'main.js')
        );
        const styleUri = webview.asWebviewUri(
            vscode.Uri.joinPath(this._extensionUri, 'media', 'main.css')
        );
        const cytoscapeUri = webview.asWebviewUri(
            vscode.Uri.joinPath(this._extensionUri, 'media', 'cytoscape.min.js')
        );

        const nonce = this._getNonce();

        return `<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="Content-Security-Policy" content="default-src 'none'; style-src ${webview.cspSource} 'unsafe-inline'; script-src 'nonce-${nonce}';">
    <link href="${styleUri}" rel="stylesheet">
    <title>Code Visualizer</title>
</head>
<body>
    <!-- Input Section -->
    <div id="input-section" class="section">
        <h3>Repository</h3>
        <div class="input-group">
            <input type="text" id="repoUrl" placeholder="https://github.com/user/repo" />
            <button id="cloneBtn" class="primary-btn">Clone & Visualize</button>
        </div>
        <div id="status-message" class="status"></div>
    </div>

    <!-- Visualization Controls -->
    <div id="controls-section" class="section">
        <h3>Visualization</h3>
        <select id="layout-select">
            <option value="tree">Tree (Breadth-First)</option>
            <option value="radial">Radial Tree</option>
            <option value="hierarchical">Hierarchical</option>
            <option value="forceDirected">Force-Directed (Physics)</option>
            <option value="circular">Circular</option>
            <option value="grid">Grid</option>
            <option value="concentric">Concentric Circles</option>
            <option value="dag">DAG</option>
        </select>
        <div class="button-row">
            <button id="refreshBtn" class="secondary-btn" title="Refresh visualization">↻ Refresh</button>
            <button id="fitBtn" class="secondary-btn" title="Fit to view">⊡ Fit</button>
        </div>
    </div>

    <!-- Legend -->
    <div id="legend-section" class="section">
        <h3>Legend</h3>
        <div class="legend">
            <div class="legend-item">
                <span class="legend-color folder"></span>
                <span>Folder</span>
            </div>
            <div class="legend-item">
                <span class="legend-color file"></span>
                <span>File</span>
            </div>
            <div class="legend-item">
                <span class="legend-color class"></span>
                <span>Class</span>
            </div>
            <div class="legend-item">
                <span class="legend-color function"></span>
                <span>Function</span>
            </div>
        </div>
    </div>

    <!-- Graph Container -->
    <div id="cy-container" class="section graph-section">
        <div id="cy"></div>
        <div id="empty-state">
            <p>Enter a GitHub repository URL above to visualize its structure</p>
        </div>
    </div>

    <!-- Node Info Panel -->
    <div id="info-panel" class="section">
        <h3>Node Details</h3>
        <div id="info-content">
            <p class="placeholder">Click a node to see details</p>
        </div>
        <button id="showCodeBtn" class="primary-btn" style="display: none;">Show Code</button>
    </div>

    <!-- Context Menu -->
    <div id="context-menu" class="context-menu">
        <div class="context-menu-item" data-action="showCode">Show Code</div>
        <div class="context-menu-item" data-action="jumpToCode">Jump to Definition</div>
        <div class="context-menu-item" data-action="highlight">Highlight Connected</div>
        <div class="context-menu-item" data-action="collapse">Collapse Children</div>
    </div>

    <script nonce="${nonce}" src="${cytoscapeUri}"></script>
    <script nonce="${scriptUri}"></script>
</body>
</html>`;
    }
```

**Security Note:** The `nonce` (number used once) is a security feature that ensures only scripts from the extension can run in the webview.

---

### Step 3: User Enters Repository URL

**File:** `media/main.js`

When the user clicks "Clone & Visualize":

```javascript
// In main.js (simplified)
document.getElementById('cloneBtn').addEventListener('click', () => {
    const repoUrl = document.getElementById('repoUrl').value;
    vscode.postMessage({
        command: 'cloneRepo',
        repoUrl: repoUrl
    });
});
```

**Message Flow:**
```
Webview (main.js) 
    → postMessage({command: 'cloneRepo', repoUrl: '...'})
    → Extension (visualizationPanel.ts)
    → _handleMessage() receives it
```

---

### Step 4: Extension Handles Clone Request

**File:** `src/visualizationPanel.ts`

```110:181:src/visualizationPanel.ts
    private async _cloneAndVisualize(repoUrl: string) {
        if (!repoUrl || !repoUrl.trim()) {
            vscode.window.showErrorMessage('Please enter a valid GitHub repository URL');
            this._sendMessage({ command: 'cloneError', error: 'Invalid URL' });
            return;
        }

        // Normalize URL
        repoUrl = repoUrl.trim();
        if (!repoUrl.startsWith('http://') && !repoUrl.startsWith('https://')) {
            repoUrl = `https://${repoUrl}`;
        }
        if (!repoUrl.includes('github.com')) {
            vscode.window.showErrorMessage('Please enter a valid GitHub repository URL');
            this._sendMessage({ command: 'cloneError', error: 'Must be a GitHub URL' });
            return;
        }

        const storagePath = this._context.globalStorageUri.fsPath;
        if (!fs.existsSync(storagePath)) {
            fs.mkdirSync(storagePath, { recursive: true });
        }

        const repoName = repoUrl.split('/').pop()?.replace('.git', '') || 'repo';
        const localRepoPath = path.join(storagePath, repoName);

        await vscode.window.withProgress({
            location: vscode.ProgressLocation.Notification,
            title: "Cloning & Parsing Repository...",
            cancellable: false
        }, async (progress) => {
            try {
                const simpleGit = (await import('simple-git')).default;
                const git = simpleGit();

                progress.report({ message: 'Cloning repository...' });

                if (!fs.existsSync(localRepoPath)) {
                    await git.clone(repoUrl, localRepoPath);
                } else {
                    try {
                        await git.cwd(localRepoPath).pull();
                    } catch (e) {
                        console.log('Pull failed, using existing repo');
                    }
                }

                progress.report({ message: 'Parsing repository structure...' });

                const parser = new RepoParser(localRepoPath);
                this._repoData = await parser.parse();
                this._localRepoPath = localRepoPath;

                // Setup file watcher
                this._setupWatcher();

                // Send data to webview
                this._sendMessage({
                    command: 'updateGraph',
                    data: this._repoData
                });

                this._sendMessage({ command: 'cloneSuccess' });
                vscode.window.showInformationMessage(`Repository visualized: ${repoName}`);

            } catch (error: any) {
                console.error('Clone error:', error);
                vscode.window.showErrorMessage(`Failed to clone repository: ${error.message}`);
                this._sendMessage({ command: 'cloneError', error: error.message });
            }
        });
    }
```

**What happens:**
1. **Validates URL**: Checks if it's a valid GitHub URL
2. **Creates storage path**: Uses VS Code's global storage directory
3. **Shows progress**: Uses `withProgress()` to show a progress notification
4. **Clones repository**: Uses `simple-git` library to clone (or pull if exists)
5. **Parses repository**: Creates a `RepoParser` instance and parses the code
6. **Sets up file watcher**: Watches for file changes to auto-update
7. **Sends data to webview**: Sends the parsed data back to the webview

**Key Points:**
- Uses VS Code's `globalStorageUri` for persistent storage
- Shows progress notifications to the user
- Handles errors gracefully with user-friendly messages

---

### Step 5: Repository Parsing

**File:** `src/repoParser.ts`

This is where the magic happens - converting code into graph data:

```19:35:src/repoParser.ts
    public async parse(): Promise<RepoStructure> {
        this.nodes = [];
        this.edges = [];
        const rootId = 'ROOT';

        this.nodes.push({
            id: rootId,
            label: path.basename(this.rootPath),
            type: 'folder',
            filePath: this.rootPath,
            lineNumber: 0,
            parentId: null
        });

        await this.scanDirectory(this.rootPath, rootId);
        return { nodes: this.nodes, edges: this.edges };
    }
```

**The parsing process:**

#### 5.1: Directory Scanning

```37:100:src/repoParser.ts
    private async scanDirectory(currentPath: string, parentId: string) {
        let items: string[] = [];
        try {
            items = fs.readdirSync(currentPath);
        } catch (e) {
            return;
        }

        for (const item of items) {
            const fullPath = path.join(currentPath, item);
            const relativePath = path.relative(this.rootPath, fullPath);

            // Ignore hidden files, __pycache__, node_modules, .git
            if (item.startsWith('.') ||
                item === '__pycache__' ||
                item === 'node_modules' ||
                item === '.git' ||
                item === '__init__.py') {
                continue;
            }

            let stats;
            try {
                stats = fs.statSync(fullPath);
            } catch (e) {
                continue;
            }

            if (stats.isDirectory()) {
                const folderId = `dir:${relativePath}`;
                this.nodes.push({
                    id: folderId,
                    label: item,
                    type: 'folder',
                    filePath: fullPath,
                    lineNumber: 0,
                    parentId: parentId
                });
                this.edges.push({
                    source: parentId,
                    target: folderId,
                    relationship: 'contains'
                });
                await this.scanDirectory(fullPath, folderId);

            } else if (item.endsWith('.py')) {
                const fileId = `file:${relativePath}`;
                this.nodes.push({
                    id: fileId,
                    label: item,
                    type: 'file',
                    filePath: fullPath,
                    lineNumber: 0,
                    parentId: parentId
                });
                this.edges.push({
                    source: parentId,
                    target: fileId,
                    relationship: 'contains'
                });
                this.parsePythonFile(fullPath, fileId);
            }
        }
    }
```

**What it does:**
1. Reads directory contents
2. Filters out ignored files/folders (`.git`, `node_modules`, etc.)
3. For directories: Creates a folder node and recursively scans
4. For Python files: Creates a file node and parses the file

#### 5.2: Python File Parsing

```102:168:src/repoParser.ts
    private parsePythonFile(filePath: string, fileNodeId: string) {
        let content: string;
        try {
            content = fs.readFileSync(filePath, 'utf-8');
        } catch (e) {
            return;
        }

        const lines = content.split('\n');
        let scopeStack = [{ indent: -1, id: fileNodeId, endLine: lines.length }];

        lines.forEach((line, index) => {
            const trimmed = line.trim();
            if (!trimmed || trimmed.startsWith('#')) return;

            const match = trimmed.match(/^(class|def)\s+([a-zA-Z_][a-zA-Z0-9_]*)/);
            if (match) {
                const typeKeyword = match[1];
                const name = match[2];
                const currentIndent = getIndentation(line);

                // Pop stack to find parent
                while (scopeStack.length > 1 && currentIndent <= scopeStack[scopeStack.length - 1].indent) {
                    scopeStack.pop();
                }

                const parentNode = scopeStack[scopeStack.length - 1];
                const nodeId = `${filePath}:${index}:${name}`;

                // Find the end line of this definition
                let endLine = lines.length - 1;
                for (let i = index + 1; i < lines.length; i++) {
                    const nextLine = lines[i];
                    if (nextLine.trim() && !nextLine.trim().startsWith('#')) {
                        const nextIndent = getIndentation(nextLine);
                        if (nextIndent <= currentIndent) {
                            endLine = i - 1;
                            break;
                        }
                    }
                }

                // Extract code snippet (first few lines)
                const snippetLines = lines.slice(index, Math.min(index + 10, endLine + 1));
                const codeSnippet = snippetLines.join('\n');

                this.nodes.push({
                    id: nodeId,
                    label: name,
                    type: typeKeyword === 'class' ? 'class' : 'function',
                    filePath: filePath,
                    lineNumber: index,
                    endLineNumber: endLine,
                    parentId: parentNode.id,
                    codeSnippet: codeSnippet
                });

                this.edges.push({
                    source: parentNode.id,
                    target: nodeId,
                    relationship: 'contains'
                });

                scopeStack.push({ indent: currentIndent, id: nodeId, endLine: endLine });
            }
        });
    }
```

**How it works:**
1. **Reads file content**: Loads the Python file
2. **Uses scope stack**: Tracks indentation levels to determine parent-child relationships
3. **Regex matching**: Finds `class` and `def` keywords
4. **Calculates end lines**: Finds where each class/function ends
5. **Creates nodes**: Adds class/function nodes to the graph
6. **Creates edges**: Links them to their parent (file, class, or function)

**Example:**
```python
# file.py
class MyClass:
    def method1(self):
        pass
    
    def method2(self):
        pass
```

Creates:
- Node: `MyClass` (type: class, parent: file.py)
- Node: `method1` (type: function, parent: MyClass)
- Node: `method2` (type: function, parent: MyClass)

---

### Step 6: Data Structure

**File:** `src/types.ts`

The parsed data is structured as:

```1:29:src/types.ts
export type NodeType = 'folder' | 'file' | 'class' | 'function';

export interface CodeNode {
    id: string;
    label: string;
    type: NodeType;
    filePath: string;
    lineNumber: number;
    endLineNumber?: number;
    parentId: string | null;
    codeSnippet?: string;
}

export interface RepoStructure {
    nodes: CodeNode[];
    edges: { source: string; target: string; relationship: string }[];
}

export interface VisualizationOption {
    id: string;
    name: string;
    description: string;
    enabled: boolean;
}

export interface VisualizationConfig {
    layoutName: string;
    options: Record<string, any>;
    style?: Record<string, any>[];
}
```

**Graph Structure:**
```
ROOT (folder)
  └── src/ (folder)
      └── main.py (file)
          └── MyClass (class)
              ├── method1 (function)
              └── method2 (function)
```

Each relationship becomes an edge: `{source: 'parent', target: 'child', relationship: 'contains'}`

---

### Step 7: Sending Data to Webview

**File:** `src/visualizationPanel.ts`

```340:352:src/visualizationPanel.ts
    private _sendMessage(message: any) {
        if (!this._webviewReady) {
            this._messageQueue.push(message);
            return;
        }
        this._sendMessageInternal(message);
    }

    private _sendMessageInternal(message: any) {
        if (this._view) {
            this._view.webview.postMessage(message);
        }
    }
```

**Message Queue Pattern:**
- If webview isn't ready, messages are queued
- Once webview sends `webviewReady`, queued messages are sent
- Prevents losing messages sent before webview initialization

**Message Format:**
```javascript
{
    command: 'updateGraph',
    data: {
        nodes: [...],
        edges: [...]
    }
}
```

---

### Step 8: Webview Receives Data

**File:** `media/main.js`

```javascript
// Listen for messages from extension
window.addEventListener('message', event => {
    const message = event.data;
    
    switch (message.command) {
        case 'updateGraph':
            initializeCytoscape(message.data);
            break;
        // ... other cases
    }
});
```

---

### Step 9: Cytoscape Graph Initialization

**File:** `media/main.js`

```245:299:media/main.js
// Initialize Cytoscape with graph data
function initializeCytoscape(graphData) {
    console.log('[Webview] Initializing Cytoscape with', graphData.nodes.length, 'nodes and', graphData.edges.length, 'edges');

    const container = document.getElementById('cy');
    const emptyState = document.getElementById('empty-state');

    if (!graphData.nodes || graphData.nodes.length === 0) {
        if (emptyState) emptyState.style.display = 'block';
        return;
    }

    if (emptyState) emptyState.style.display = 'none';

    const nodes = graphData.nodes.map(n => ({
        data: {
            id: n.id,
            label: n.label,
            type: n.type,
            filePath: n.filePath,
            line: n.lineNumber,
            endLine: n.endLineNumber,
            parentId: n.parentId
        }
    }));

    const edges = graphData.edges.map((e, i) => ({
        data: {
            id: 'e' + i,
            source: e.source,
            target: e.target,
            relationship: e.relationship || 'contains'
        }
    }));

    // Destroy existing instance
    if (cy) {
        cy.destroy();
        cy = null;
    }

    cy = cytoscape({
        container: container,
        elements: [...nodes, ...edges],
        style: defaultStyles,
        layout: {
            name: 'breadthfirst',
            directed: true,
            padding: 30,
            spacingFactor: 1.5,
            avoidOverlap: true,
            animate: false
        },
        wheelSensitivity: 0.3,
        minZoom: 0.1,
```

**What happens:**
1. **Transforms data**: Converts nodes/edges to Cytoscape format
2. **Destroys old graph**: Clears previous visualization
3. **Creates new graph**: Initializes Cytoscape with data
4. **Applies styles**: Uses default styles (colors, shapes, etc.)

**Cytoscape Format:**
```javascript
{
    data: {
        id: 'node1',
        label: 'MyClass',
        type: 'class',
        filePath: '/path/to/file.py',
        line: 5
    }
}
```

---

### Step 10: Visualization Layouts

**File:** `src/visualizations/index.ts`

The extension supports multiple layout algorithms:

```12:96:src/visualizations/index.ts
// Registry of all available visualizations
export const visualizationRegistry: Map<string, {
    option: VisualizationOption;
    getConfig: () => VisualizationConfig;
}> = new Map();

// Register all built-in visualizations
visualizationRegistry.set('tree', {
    option: {
        id: 'tree',
        name: 'Tree (Breadth-First)',
        description: 'Hierarchical tree layout showing parent-child relationships',
        enabled: true
    },
    getConfig: getTreeLayout
});

visualizationRegistry.set('radial', {
    option: {
        id: 'radial',
        name: 'Radial Tree',
        description: 'Circular tree with root at center, branches radiating outward',
        enabled: true
    },
    getConfig: getRadialLayout
});

visualizationRegistry.set('hierarchical', {
    option: {
        id: 'hierarchical',
        name: 'Hierarchical',
        description: 'Top-down hierarchical layout with levels',
        enabled: true
    },
    getConfig: getHierarchicalLayout
});

visualizationRegistry.set('forceDirected', {
    option: {
        id: 'forceDirected',
        name: 'Force-Directed (Physics)',
        description: 'Physics-based layout where nodes repel and edges attract',
        enabled: true
    },
    getConfig: getForceDirectedLayout
});

visualizationRegistry.set('circular', {
    option: {
        id: 'circular',
        name: 'Circular',
        description: 'All nodes arranged in a circle',
        enabled: true
    },
    getConfig: getCircularLayout
});

visualizationRegistry.set('grid', {
    option: {
        id: 'grid',
        name: 'Grid',
        description: 'Nodes arranged in a grid pattern',
        enabled: true
    },
    getConfig: getGridLayout
});

visualizationRegistry.set('concentric', {
    option: {
        id: 'concentric',
        name: 'Concentric Circles',
        description: 'Nodes arranged in concentric circles by type',
        enabled: true
    },
    getConfig: getConcentricLayout
});

visualizationRegistry.set('dag', {
    option: {
        id: 'dag',
        name: 'DAG (Directed Acyclic Graph)',
        description: 'Optimized layout for directed graphs',
        enabled: true
    },
    getConfig: getDagLayout
});
```

**Example Layout Config:**

```3:101:src/visualizations/treeLayout.ts
export function getTreeLayout(): VisualizationConfig {
    return {
        layoutName: 'breadthfirst',
        options: {
            directed: true,
            padding: 30,
            spacingFactor: 1.5,
            avoidOverlap: true,
            nodeDimensionsIncludeLabels: true,
            animate: true,
            animationDuration: 500,
            animationEasing: 'ease-out-cubic',
            roots: undefined, // Will be set to root node
            maximal: false
        },
        style: [
            {
                selector: 'node',
                style: {
                    'background-color': '#4a9eff',
                    'label': 'data(label)',
                    'color': '#ffffff',
                    'text-valign': 'center',
                    'text-halign': 'center',
                    'font-size': '11px',
                    'font-weight': 'bold',
                    'width': 'label',
                    'height': 'label',
                    'padding': '12px',
                    'shape': 'round-rectangle',
                    'border-width': '2px',
                    'border-color': '#2d7ad6',
                    'text-wrap': 'wrap',
                    'text-max-width': '100px'
                }
            },
            {
                selector: 'node[type="folder"]',
                style: {
                    'background-color': '#f5a623',
                    'border-color': '#d4850b',
                    'shape': 'rectangle'
                }
            },
            {
                selector: 'node[type="file"]',
                style: {
                    'background-color': '#7ed321',
                    'border-color': '#5ba018',
                    'shape': 'round-rectangle'
                }
            },
            {
                selector: 'node[type="class"]',
                style: {
                    'background-color': '#bd10e0',
                    'border-color': '#8a0ba3',
                    'shape': 'diamond',
                    'width': '80px',
                    'height': '50px'
                }
            },
            {
                selector: 'node[type="function"]',
                style: {
                    'background-color': '#50e3c2',
                    'border-color': '#2db89e',
                    'shape': 'ellipse',
                    'color': '#1a1a1a'
                }
            },
            {
                selector: 'edge',
                style: {
                    'width': 2,
                    'line-color': '#666666',
                    'target-arrow-color': '#666666',
                    'target-arrow-shape': 'triangle',
                    'curve-style': 'bezier',
                    'arrow-scale': 1.2
                }
            },
            {
                selector: 'node:selected',
                style: {
                    'border-width': '4px',
                    'border-color': '#ff6b6b',
                    'box-shadow': '0 0 10px #ff6b6b'
                }
            },
            {
                selector: 'node.highlighted',
                style: {
                    'background-color': '#ff6b6b',
                    'border-color': '#ff3b3b'
                }
            }
        ]
    };
}
```

**Layout Types:**
- **Tree**: Hierarchical top-down
- **Radial**: Root at center, branches outward
- **Force-Directed**: Physics simulation (nodes repel, edges attract)
- **Circular**: All nodes in a circle
- **Grid**: Organized grid pattern
- **Concentric**: Circles by node type
- **DAG**: Optimized for directed graphs

---

### Step 11: User Interactions

**File:** `media/main.js`

When user clicks a node:

```javascript
cy.on('tap', 'node', (evt) => {
    const node = evt.target;
    selectedNode = node;
    
    // Show node info
    showNodeInfo(node.data());
    
    // Request code from extension
    vscode.postMessage({
        command: 'requestNodeCode',
        nodeId: node.id()
    });
});
```

**File:** `src/visualizationPanel.ts`

Extension handles the request:

```285:318:src/visualizationPanel.ts
    private async _sendNodeCode(nodeId: string) {
        if (!this._repoData) return;

        const node = this._repoData.nodes.find(n => n.id === nodeId);
        if (!node) return;

        try {
            const content = fs.readFileSync(node.filePath, 'utf-8');
            const lines = content.split('\n');

            let codeSnippet: string;
            if (node.type === 'file' || node.type === 'folder') {
                codeSnippet = lines.slice(0, 50).join('\n');
                if (lines.length > 50) {
                    codeSnippet += '\n... (truncated)';
                }
            } else {
                const startLine = node.lineNumber;
                const endLine = node.endLineNumber || Math.min(startLine + 20, lines.length - 1);
                codeSnippet = lines.slice(startLine, endLine + 1).join('\n');
            }

            this._sendMessage({
                command: 'nodeCode',
                nodeId: nodeId,
                code: codeSnippet,
                filePath: node.filePath,
                startLine: node.lineNumber,
                endLine: node.endLineNumber
            });
        } catch (error) {
            console.error('Error reading code:', error);
        }
    }
```

**When user clicks "Show Code":**

```237:264:src/visualizationPanel.ts
    private async _showCode(nodeData: any) {
        if (!nodeData || !nodeData.filePath) return;

        try {
            const doc = await vscode.workspace.openTextDocument(nodeData.filePath);
            const editor = await vscode.window.showTextDocument(doc, {
                viewColumn: vscode.ViewColumn.One,
                preserveFocus: false
            });

            const startLine = nodeData.line || 0;
            const endLine = nodeData.endLine || startLine;

            // Highlight the relevant code
            const startPos = new vscode.Position(startLine, 0);
            const endPos = new vscode.Position(endLine, doc.lineAt(Math.min(endLine, doc.lineCount - 1)).text.length);

            editor.selection = new vscode.Selection(startPos, endPos);
            editor.revealRange(
                new vscode.Range(startPos, endPos),
                vscode.TextEditorRevealType.InCenterIfOutsideViewport
            );

        } catch (error) {
            console.error('Error showing code:', error);
            vscode.window.showErrorMessage(`Could not open file: ${nodeData.filePath}`);
        }
    }
```

**What happens:**
1. Opens the file in VS Code editor
2. Highlights the relevant code section
3. Scrolls to center the code in view

---

### Step 12: File Watching

**File:** `src/visualizationPanel.ts`

The extension watches for file changes:

```183:228:src/visualizationPanel.ts
    private _setupWatcher() {
        this._disposeWatcher();

        if (!this._localRepoPath) return;

        let updateTimeout: NodeJS.Timeout | undefined;

        const handleFileChange = async (filePath: string) => {
            if (!filePath.endsWith('.py')) return;
            if (!this._localRepoPath || !filePath.startsWith(this._localRepoPath)) return;

            console.log(`[Watcher] Change detected: ${filePath}`);
            clearTimeout(updateTimeout);

            updateTimeout = setTimeout(async () => {
                vscode.window.setStatusBarMessage(`Re-parsing: ${path.basename(filePath)}`, 3000);

                try {
                    const parser = new RepoParser(this._localRepoPath!);
                    this._repoData = await parser.parse();

                    this._sendMessage({
                        command: 'updateGraph',
                        data: this._repoData
                    });
                } catch (error) {
                    console.error('Re-parse error:', error);
                }
            }, 500);
        };

        try {
            this._watcher = fs.watch(this._localRepoPath, {
                recursive: true,
                encoding: 'utf8'
            }, (eventType, filename) => {
                if (!filename || !this._localRepoPath) return;
                const fullPath = path.resolve(this._localRepoPath, filename);
                handleFileChange(fullPath);
            });

            console.log(`[Watcher] Watching: ${this._localRepoPath}`);
        } catch (error) {
            console.error('Watcher setup failed:', error);
        }
    }
```

**Features:**
- **Debouncing**: 500ms delay to avoid excessive re-parsing
- **Recursive watching**: Watches entire directory tree
- **Auto-update**: Graph updates automatically when files change
- **Status messages**: Shows "Re-parsing..." in status bar

---

## 📊 Component Breakdown

### 1. Extension Entry Point (`extension.ts`)

**Responsibilities:**
- Activates extension
- Registers webview provider
- Registers commands
- Manages extension lifecycle

**Key Functions:**
- `activate()`: Called when extension loads
- `deactivate()`: Called when extension unloads

---

### 2. Visualization Panel (`visualizationPanel.ts`)

**Responsibilities:**
- Manages w