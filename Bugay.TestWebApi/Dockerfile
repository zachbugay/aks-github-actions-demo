# Use the official .NET SDK image for building
FROM mcr.microsoft.com/dotnet/nightly/sdk:10.0-preview AS build
WORKDIR /app

# Copy csproj and restore dependencies
COPY *.csproj ./
RUN dotnet restore

# Copy everything else and build
COPY . ./
RUN dotnet publish -c Release -o out

# Build runtime image
FROM mcr.microsoft.com/dotnet/nightly/aspnet:10.0-preview
WORKDIR /app
COPY --from=build /app/out .

# Expose port
EXPOSE 5028
ENV ASPNETCORE_URLS=http://+:5028

ENTRYPOINT ["dotnet", "Bugay.TestWebApi.dll"]