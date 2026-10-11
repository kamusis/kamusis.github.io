---
title: >-
  Setting Up PostgreSQL Development Environment with VS Code, DevContainer, and
  Windsurf
date: 2025-08-31 17:43:04
tags: [PostgreSQL, VSCode, DevContainer, Windsurf, Development Setup, AI Coding]
---

Developing PostgreSQL from source on Windows can be challenging due to the need for numerous build tools and dependencies. Using a development container (DevContainer) provides a consistent, isolated environment that works seamlessly across Windows, macOS, and Linux, eliminating platform-specific setup hassles.

Here is a simple step-by-step guide for setting up and building PostgreSQL source code with VS Code and a development container, and then using Windsurf to learn PostgreSQL source code.

## Setup and Build PostgreSQL in VS Code

### 1. Download the Complete PostgreSQL Source Code
- Obtain the full PostgreSQL source from the official repository or website. Typically run:
```bash
git clone https://git.postgresql.org/git/postgresql.git
```

### 2. Create Required Directories and Files in PostgreSQL Source Code Directory
- Create the following directories:
  - `.vscode`
  - `.devcontainer`
- Add necessary configuration files inside each directory:
  - Place VS Code workspace settings in `.vscode`
  - Add development container configuration files (e.g., `devcontainer.json` and `Dockerfile`) in `.devcontainer`.

Add the following content to the `devcontainer.json` file:
```json
{
  "name": "PostgreSQL Dev",
  "build": {
    "dockerfile": "Dockerfile"
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-vscode.cpptools",
        "ms-vscode.cmake-tools",
        "Codeium.codeium"
      ],
      "settings": {
        "editor.formatOnSave": true,
        "files.eol": "\n",
        "editor.codeActionsOnSave": {
          "source.fixAll.eslint": "explicit"
        }
      }
    }
  }
}
```

Add the following content to the `Dockerfile` file:
```Dockerfile
FROM ubuntu:22.04

RUN apt update && apt install -y \
    build-essential \
    flex \
    bison \
    libreadline-dev \
    zlib1g-dev \
    pkg-config \
    libssl-dev \
    libxml2-dev \
    libxslt1-dev \
    libedit-dev \
    libicu-dev \
    git \
    curl \
    && rm -rf /var/lib/apt/lists/*

ENV CC=/usr/bin/gcc
ENV CXX=/usr/bin/g++
```
- For editors using the Microsoft C/C++ extension, it’s recommended to add a `c_cpp_properties.json` file to the `.vscode`.
```json
{
  "configurations": [
    {
      "name": "Linux",
      "includePath": [
        "${workspaceFolder}/**",
        "${workspaceFolder}/src/include",
        "${workspaceFolder}/src/include/utils",
        "${workspaceFolder}/src/backend",
        "${workspaceFolder}/src/backend/utils",
        "/usr/include",
        "/usr/local/include"
      ],
      "defines": [],
      "compilerPath": "/usr/bin/gcc",
      "cStandard": "c11",
      "cppStandard": "c++14",
      "intelliSenseMode": "linux-gcc-x64"
    }
  ],
  "version": 4
}
```

- For editors (e.g., Windsurf) using clangd instead of the Microsoft C/C++ extension, it’s recommended to add a `.clangd` configuration file and a `compile_commands.json` file to the project root.

<u>.clangd</u>

```
CompileFlags:
  Add: [
    "-I${workspaceFolder}/src/include",
    "-I${workspaceFolder}/src/include/utils",
    "-I${workspaceFolder}/src/backend", 
    "-I${workspaceFolder}/src/backend/utils",
    "-std=c11"
  ]
```

<u>compile_commands.json</u>

```json
[
  {
    "directory": "/workspaces/postgresql",
    "command": "/usr/bin/gcc -I/workspaces/postgresql/src/include -I/workspaces/postgresql/src/backend -c /workspaces/postgresql/src/backend/bootstrap/bootstrap.c -o bootstrap.o",
    "file": "/workspaces/postgresql/src/backend/bootstrap/bootstrap.c"
  },
  {
    "directory": "/workspaces/postgresql",
    "command": "/usr/bin/gcc -I/workspaces/postgresql/src/include -I/workspaces/postgresql/src/backend -c /workspaces/postgresql/src/backend/utils/init/globals.c -o globals.o",
    "file": "/workspaces/postgresql/src/backend/utils/init/globals.c"
  }
]
```

- To ensure consistent line endings and proper handling of text and binary files in your PostgreSQL project, add the following content to the`.gitattributes` file in project root:

```gitattributes
# Set default behavior to automatically normalize line endings to LF
* text=auto eol=lf

# Explicitly declare text files to be normalized to LF
*.c text eol=lf
*.h text eol=lf
*.cpp text eol=lf
*.hpp text eol=lf
*.cc text eol=lf
*.hh text eol=lf
*.py text eol=lf
*.sh text eol=lf
*.pl text eol=lf
*.pm text eol=lf
*.sql text eol=lf
Makefile text eol=lf
makefile text eol=lf
*.mk text eol=lf

# Declare binary files that should not be modified
*.png binary
*.jpg binary
*.gif binary
*.ico binary
*.zip binary
*.tar binary
*.gz binary
*.pdf binary
```

This configuration will automatically normalize line endings for text files to LF, and protect binary files from unwanted line ending conversions, improving cross-platform compatibility.

Finnally, the newly created directories and files and modified files should look like this:

```
postgresql/
├── All the stuff in the original postgresql source code
├── .devcontainer/
│   ├── devcontainer.json
│   └── Dockerfile
├── .vscode/
│   └── c_cpp_properties.json
├── .gitattributes
├── .clangd
└── compile_commands.json
```

### 3. Reopen Folder in Container (VS Code)
- In VS Code, use the "Dev Containers: Reopen in Container" command to open your workspace within the defined development container. If you can't find this command by Ctrl+Shift+P (or Cmd+Shift+P on macOS), you can install the Dev Containers extension from the VS Code marketplace.

### 4. Build PostgreSQL in the Container
- In the container's terminal, execute:
  ```
  ./configure && make
  ```
- This will configure the build and compile all required files, including generated headers such as `errcodes.h`.

These steps ensure a stable environment for building and developing PostgreSQL efficiently with VS Code and containers.

- ### Access this Devcontainer from Windsurf
  - Close VS Code, the devcontainer will also stop automatically. No way to keep it running.
  - Use `docker ps -a` to find the container id of this devcontainer.
  - Use `docker start <container_id>` to start the devcontainer.
  - Open Windsurf, use `Open a Remote Window -> Attach to Running Container` to attach to this devcontainer.
  - Windsurf cannot use Microsoft C/C++ extension anymore, use clangd instead. Install clangd extension in Windsurf.
  - In Windsurf, open the postgresql source code directory, should be `/workspaces/postgresql`.

  ![20250831181535](https://s2.loli.net/2025/08/31/qxi1N8VKfdtQ9D3.png)

  Thanks to the Cascade and the latest feature - DeepWiki of Windsurf, you can now enjoy the brand new learning experience powered by AI.
