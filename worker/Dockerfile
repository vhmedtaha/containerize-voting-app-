FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build 

WORKDIR /src

COPY . .

RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS runtime

WORKDIR /app

COPY --from=build /app/publish .

CMD ["dotnet", "Worker.dll"]