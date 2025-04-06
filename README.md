# Debugging Docker Containers in Visual Studio Code

This documentation explains how to set up and debug a .NET application running inside a Docker container using Visual Studio Code. The example provided is for a .NET 9.0 Web API project (`docker.api`) running in a Docker container.

---

## Prerequisites

1. **Docker Installed**: Ensure Docker is installed and running on your machine.
2. **Visual Studio Code**: Install VS Code with the following extensions:
   - **C#**: For .NET development.
   - **Docker**: For Docker integration.
3. **.NET SDK**: Install the .NET 9.0 SDK on your host machine.
4. **Debugger Path**: Ensure the `.vsdbg` debugger is installed and accessible.

---

## Project Structure

The project structure includes the following key files:

- **Dockerfile**: Defines the Docker image for the application.
- **`tasks.json`**: Contains tasks for building and running the Docker container.
- **`launch.json`**: Configures the debugger to attach to the running container.
- **`Program.cs`**: Entry point for the .NET application.
- **`docker-compose.yml`**: Optional file for managing multiple services.

---

## Key Configuration Files

### 1. **Dockerfile**
The Dockerfile defines the steps to build the Docker image, including installing the debugger (`vsdbg`) and publishing the application.

```dockerfile
# Use the ASP.NET runtime image as the base
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS base
WORKDIR /app
EXPOSE 5264
ENV ASPNETCORE_URLS=http://+:5264
ENV ASPNETCORE_ENVIRONMENT=Development

# Install the debugger
RUN apt-get update \
    && apt-get install -y --no-install-recommends unzip curl \
    && mkdir -p /remote_debugger \
    && curl -sSL https://aka.ms/getvsdbgsh -o /tmp/getvsdbg.sh \
    && bash /tmp/getvsdbg.sh -v latest -l /remote_debugger \
    && rm /tmp/getvsdbg.sh \
    && chmod -R 755 /remote_debugger

# Use the .NET SDK image for building the application
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src
COPY ["docker.api/docker.api.csproj", "docker.api/"]
RUN dotnet restore "docker.api/docker.api.csproj"
COPY . .
WORKDIR "/src/docker.api"
RUN dotnet build "docker.api.csproj" -c Debug -o /app/build

# Publish the application in Debug mode
FROM build AS publish
WORKDIR "/src/docker.api"
RUN dotnet publish "docker.api.csproj" -c Debug -o /app/publish /p:UseAppHost=false

# Final image with the runtime and application
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "docker.api.dll"]
```

---

### 2. **tasks.json**
The tasks.json file defines tasks for building and running the Docker container.

```jsonc
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "docker-run: debug",
      "type": "shell",
      "command": "docker",
      "args": [
        "run",
        "-d",
        "--name",
        "docker.api.debug",
        "-p",
        "5264:5264",
        "--env",
        "ASPNETCORE_ENVIRONMENT=Development",
        "--env",
        "DOTNET_USE_POLLING_FILE_WATCHER=1",
        "docker.api:dev"
      ],
      "dependsOn": ["docker-build"],
      "problemMatcher": []
    },
    {
      "label": "docker-build",
      "type": "docker-build",
      "dockerBuild": {
        "tag": "docker.api:dev",
        "dockerfile": "${workspaceFolder}/docker.api/Dockerfile",
        "context": "${workspaceFolder}"
      }
    },
    {
      "label": "docker-stop: debug",
      "type": "shell",
      "command": "docker",
      "args": [
        "stop",
        "docker.api.debug"
      ],
      "problemMatcher": [],
      "presentation": {
        "reveal": "silent",
        "panel": "shared"
      }
    },
    {
      "label": "docker-rm: debug",
      "type": "shell",
      "command": "docker",
      "args": [
        "rm",
        "-f",
        "docker.api.debug"
      ],
      "problemMatcher": [],
      "presentation": {
        "reveal": "silent",
        "panel": "shared"
      }
    }
  ]
}
```

---

### 3. **launch.json**
The launch.json file configures the debugger to attach to the running Docker container.

```jsonc
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Docker .NET Attach",
      "type": "docker",
      "request": "attach",
      "preLaunchTask": "docker-run: debug",
      "postDebugTask": "docker-stop: debug",
      "netCore": {
        "appProject": "${workspaceFolder}/docker.api/docker.api.csproj",
        "debuggerPath": "/remote_debugger/vsdbg",
        "appPath": "/app/docker.api.dll"
      },
      "containerName": "docker.api.debug",
      "platform": "netcore",
      "pipeTransport": {
        "pipeProgram": "docker",
        "pipeArgs": [
          "exec",
          "-i",
          "docker.api.debug",
          "/remote_debugger/vsdbg",
          "--interpreter=vscode",
          "--",
          "/app/docker.api.dll"
        ],
        "debuggerPath": "/remote_debugger/vsdbg",
        "pipeCwd": "${workspaceFolder}"
      }
    }
  ]
}
```

---

## Steps to Debug

1. **Build the Docker Image**:
   Run the `docker-build` task in VS Code:
   - Open the Command Palette (`Ctrl+Shift+P`).
   - Select `Tasks: Run Task`.
   - Choose `docker-build`.

2. **Run the Docker Container**:
   Run the `docker-run: debug` task:
   - Open the Command Palette.
   - Select `Tasks: Run Task`.
   - Choose `docker-run: debug`.

3. **Attach the Debugger**:
   - Open the **Run and Debug** panel (`Ctrl+Shift+D`).
   - Select the `Docker .NET Attach` configuration.
   - Press `F5` to start debugging.

---

## Common Issues and Fixes

### 1. **Debugger Path Error**
If you see an error like `"/remote_debugger/vsdbg": is a directory`, ensure the debugger is installed correctly in the container:
- Verify the debugger installation:
  ```bash
  docker exec -it docker.api.debug ls -l /remote_debugger
  ```
- Ensure the `debuggerPath` in launch.json points to `/remote_debugger/vsdbg`.

### 2. **Application Not Found**
If the application (`docker.api.dll`) is not found:
- Verify the application files are in `/app/publish`:
  ```bash
  docker exec -it docker.api.debug ls -l /app/publish
  ```
- Ensure the `appPath` in launch.json points to `/app/docker.api.dll`.

### 3. **Container Not Running**
If the container is not running:
- Check the container logs:
  ```bash
  docker logs docker.api.debug
  ```

---

## Conclusion

This setup allows you to debug a .NET application running in a Docker container using Visual Studio Code. By ensuring the debugger and application paths are correctly configured, you can seamlessly attach the debugger and troubleshoot your application.



# Dockerfile for Production


1. **Debugger Installation**: The `base` stage installs the Visual Studio Debugger (`vsdbg`) using a script from `aka.ms/getvsdbgsh`. This is useful for debugging in development but unnecessary—and potentially a security risk—in production.
2. **Environment Setting**: `ENV ASPNETCORE_ENVIRONMENT=Development` explicitly sets the environment to "Development," which typically enables detailed error messages and other development-friendly behaviors not suitable for production.
3. **Debug Configuration**: The `build` and `publish` stages use the `Debug` configuration (`-c Debug`), which includes debugging symbols and unoptimized code, making the application larger and slower than a production-optimized `Release` build.
4. **Unnecessary Tools**: The `apt-get install` commands add `unzip` and `curl` to the base image for debugger setup, increasing the image size, which isn’t needed in production.

### Why a Separate Dockerfile for Production?
- **Security**: Including a debugger or development tools in a production image could expose vulnerabilities or provide attackers with unnecessary tools if the container is compromised.
- **Performance**: A `Release` build is optimized for speed and efficiency, unlike a `Debug` build.
- **Size**: Excluding unnecessary tools (like `vsdbg`, `curl`, and `unzip`) reduces the image size, which improves deployment speed and resource usage.
- **Environment**: Production typically uses `ASPNETCORE_ENVIRONMENT=Production`, which disables development-specific features like detailed error pages.

### Example Production Dockerfile
Here’s how you could modify the Dockerfile for production:

```dockerfile
# Use the ASP.NET runtime image as the base
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS base
WORKDIR /app
EXPOSE 5264
ENV ASPNETCORE_URLS=http://+:5264
ENV ASPNETCORE_ENVIRONMENT=Production

# Use the .NET SDK image for building the application
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src
COPY ["docker.api/docker.api.csproj", "docker.api/"]
RUN dotnet restore "docker.api/docker.api.csproj"
COPY . .
WORKDIR "/src/docker.api"
RUN dotnet build "docker.api.csproj" -c Release -o /app/build

# Publish the application in Release mode
FROM build AS publish
WORKDIR "/src/docker.api"
RUN dotnet publish "docker.api.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Final image with the runtime and application
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "docker.api.dll"]
```

### Key Changes for Production
1. **Removed Debugger**: The entire `RUN apt-get update ...` block is omitted since the debugger isn’t needed.
2. **Environment**: Changed `ASPNETCORE_ENVIRONMENT` to `Production`.
3. **Build Configuration**: Switched `-c Debug` to `-c Release` in both `dotnet build` and `dotnet publish` commands for optimized output.
4. **Minimal Base Image**: The `base` stage no longer includes unnecessary packages, keeping it lean.

### Alternative Approach: Single Dockerfile with Arguments
If you prefer to maintain a single Dockerfile, you could use build arguments (`ARG`) and conditionals to toggle between development and production setups. For example:

```dockerfile
# Use the ASP.NET runtime image as the base
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS base
WORKDIR /app
EXPOSE 5264
ENV ASPNETCORE_URLS=http://+:5264
ARG ENV=Development
ENV ASPNETCORE_ENVIRONMENT=$ENV

# Install debugger only if ENV is Development
RUN if [ "$ENV" = "Development" ]; then \
    apt-get update \
    && apt-get install -y --no-install-recommends unzip curl \
    && mkdir -p /remote_debugger \
    && curl -sSL https://aka.ms/getvsdbgsh -o /tmp/getvsdbg.sh \
    && bash /tmp/getvsdbg.sh -v latest -l /remote_debugger \
    && rm /tmp/getvsdbg.sh \
    && chmod -R 755 /remote_debugger; \
    fi

# Use the .NET SDK image for building
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src
COPY ["docker.api/docker.api.csproj", "docker.api/"]
RUN dotnet restore "docker.api/docker.api.csproj"
COPY . .
WORKDIR "/src/docker.api"
ARG CONFIG=Debug
RUN dotnet build "docker.api.csproj" -c $CONFIG -o /app/build

# Publish the application
FROM build AS publish
WORKDIR "/src/docker.api"
RUN dotnet publish "docker.api.csproj" -c $CONFIG -o /app/publish /p:UseAppHost=false

# Final image
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "docker.api.dll"]
```

Then build with:
- Development: `docker build -t myapp:dev --build-arg ENV=Development --build-arg CONFIG=Debug .`
- Production: `docker build -t myapp:prod --build-arg ENV=Production --build-arg CONFIG=Release .`

### Recommendation
While the single-file approach with arguments works, maintaining separate Dockerfiles (e.g., `Dockerfile.dev` and `Dockerfile.prod`) is often clearer and less error-prone, especially for teams. It ensures that production images are explicitly optimized and free of development artifacts without relying on runtime conditions. So, yes, I’d recommend a separate Dockerfile for production to keep things clean and secure.