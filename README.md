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