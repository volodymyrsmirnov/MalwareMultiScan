FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS builder

WORKDIR /src

COPY MalwareMultiScan.Api /src/MalwareMultiScan.Api
COPY MalwareMultiScan.Backends /src/MalwareMultiScan.Backends
COPY MalwareMultiScan.Shared /src/MalwareMultiScan.Shared

RUN dotnet publish -c Release -r linux-x64 -o ./publish MalwareMultiScan.Api/MalwareMultiScan.Api.csproj

FROM mcr.microsoft.com/dotnet/core/aspnet:3.1

WORKDIR /worker

COPY --from=builder /src/publish /worker

ENV ASPNETCORE_URLS=http://+:5000
ENV ASPNETCORE_ENVIRONMENT=Production

ENTRYPOINT ["/worker/MalwareMultiScan.Api"]