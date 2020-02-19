FROM mcr.microsoft.com/dotnet/core/sdk:3.0
ENV ASPNETCORE_ENVIRONMENT=Development
WORKDIR /app
COPY . .
RUN dotnet restore
ENTRYPOINT[“dotnet”,”run”]
