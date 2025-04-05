# Use the ASP.NET runtime image as the base
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS base
WORKDIR /app
EXPOSE 5264
ENV ASPNETCORE_URLS=http://+:5264
ENV ASPNETCORE_ENVIRONMENT=Development
ENV ASPNETCORE_LOGGING__CONSOLE__DISABLECOLORS=true

# Install the debugger with explicit verification
RUN apt-get update \
    && apt-get install -y --no-install-recommends unzip curl \
    && mkdir -p /remote_debugger \
    && curl -sSL https://aka.ms/getvsdbgsh -o /tmp/getvsdbg.sh \
    && bash /tmp/getvsdbg.sh -v latest -l /remote_debugger \
    && rm /tmp/getvsdbg.sh \
    && ls -la /remote_debugger \
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