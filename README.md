- We have our code of microservices and we are deploying them using docker and below we have given details information about how we write dockerfiles and docker-compose files and also we are going to see that how we have to configure our project

 Prerequisites for our project
 
  ![alt text](https://img.shields.io/badge/Docker%20-8A2BE2)
  ![alt text](https://img.shields.io/badge/MySql%20-8A2BE2)
  ![alt text](https://img.shields.io/badge/MongoDB%20-8A2BE2)
  - MySQL server
  - MongoDB server

   
1. Clone the repository:
   - git clone [https://gitlab.orangebitsindia.com/spinoffedutech/spinoffedutech-backend.git]
   - cd spinoffedutech-backend

  
- Deployment using Docker and Docker Compose
      
      #For ApiGateway
      FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
      WORKDIR /app
      EXPOSE 80
      EXPOSE 443
      FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
      WORKDIR /src
      COPY ["ApiGateway.csproj", "ApiGateway/"]
      RUN dotnet restore "ApiGateway/ApiGateway.csproj"
      WORKDIR "/src/ApiGateway"
      COPY . .
      RUN dotnet build "ApiGateway.csproj" -c Release -o /app/build
      
      FROM build AS publish
      RUN dotnet publish "ApiGateway.csproj" -c Release -o /app/publish /p:UseAppHost=false
      
      FROM base AS final
      WORKDIR /app
      COPY --from=publish /app/publish .
      ENTRYPOINT ["dotnet", "ApiGateway.dll"]

- Describing Steps of Dockerfile we are using in this project

    - FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
       - This line sets the base image for the build stage named "base." It uses the official Microsoft ASP.NET Core runtime image for .NET 8.0. This image is optimized for running ASP.NET Core applications in production.

    - WORKDIR /app
       - This sets the working directory within the container to /app. Subsequent commands will be executed within this directory.

    - EXPOSE 80
       - This instruction informs Docker that the container will listen on port 80 at runtime. This is a documentation feature and does not actually publish the specified ports.

    - EXPOSE 443
       - Similar to the previous EXPOSE command, this one informs Docker that the container will listen on port 443.

    - FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
       - This sets up another stage named "build" using the official Microsoft SDK image for .NET 8.0. This stage is responsible for building the application.

    - WORKDIR /src
       - Changes the working directory to /src within the container.

    - COPY ["ApiGateway.csproj", "ApiGateway/"]
       - Copies the project file (ApiGateway.csproj) into the container at the path ApiGateway/. This allows Docker to take advantage of caching for the dependencies restoration step.

    - RUN dotnet restore "ApiGateway/ApiGateway.csproj"
       - Restores the NuGet dependencies for the project.

    - WORKDIR "/src/ApiGateway"
      - Changes the working directory to the actual location of the application code.

    - COPY . .
      - Copies all files from the current directory (where the Dockerfile is located) into the container at the current working directory.

    - RUN dotnet build "ApiGateway.csproj" -c Release -o /app/build
      - Builds the application in release mode and outputs the binaries to /app/build.

    - FROM build AS publish
      - Creates a new stage named "publish" based on the previous "build" stage.
      
    - RUN dotnet publish "ApiGateway.csproj" -c Release -o /app/publish /p:UseAppHost=false
      - Publishes the application to /app/publish with the specified configuration. The /p:UseAppHost=false flag indicates that the application does not use an executable host.

    - FROM base AS final
      - Sets up the final stage based on the "base" stage, which contains only the ASP.NET Core runtime.

    - WORKDIR /app
      - Changes the working directory to /app within the container.

    - COPY --from=publish /app/publish .
      - Copies the published application from the "publish" stage to the current directory in the final stage.

    - ENTRYPOINT ["dotnet", "ApiGateway.dll"]
      - Specifies the default command to run when the container starts. In this case, it runs the compiled ASP.NET Core application (ApiGateway.dll) using the dotnet command.


- Describing Docker Compose file

        services:
          db:
            image: mysql:8.3.0
            hostname: db
            restart: always
            environment:
              MYSQL_ROOT_PASSWORD: Tech8092
              MYSQL_DATABASE: edutech_account
            healthcheck:
              test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -u root --password=Tech8092"]
              interval: 5s
              timeout: 10s
              retries: 5
            ports:
              - 3306:3306
            networks:
              - spin-off-backend
            volumes:
              - db-data:/var/lib/mysql
        
          userapi:
            build:
              context: ./User.Api
              target: final
            restart: always
            environment:
              ASPNETCORE_ENVIRONMENT: Development
            ports:
              - 7057:8080
            networks:
              - spin-off-backend
            depends_on:
              db:
                condition: service_healthy
        
          apigateway:
            build:
              context: ./ApiGateway
              target: final
            restart: always
            environment:
              ASPNETCORE_ENVIRONMENT: Development
            ports:
              - 8080:8080
            networks:
              - spin-off-backend
            depends_on:
              db:
                condition: service_healthy
        
          contentapi:
            build:
              context: ./Content.Api
              target: final
            restart: always
            environment:
              ASPNETCORE_ENVIRONMENT: Development
            ports:
              - 8090:8080
            networks:
              - spin-off-backend
            depends_on:
              db:
                condition: service_healthy
        
        
          mongodb:
            image: mongo:latest
            container_name: edutech_mongodb
            hostname: localhost
            command: >
              bash -c "apt-get update && apt-get install -y && exec mongod --bind_ip_all"
            environment:
              MONGO_INITDB_DATABASE: EdutechContentDB
            ports:
              - "27017:27017"
            volumes:
              - mongodb_data:/data/db
            networks:
              - spin-off-backend
            healthcheck:
              test: ["CMD-SHELL", "mongosh --eval 'db.runCommand({ ping: 1 })'"]
              interval: 10s
              timeout: 50s  # Increase the timeout if necessary
              retries: 6
        
        
        networks:
          spin-off-backend:
        
        volumes:
          db-data:
          mongodb_data:



    - MySQL Service (db):
        Image: Pulls the MySQL 8.3.0 image from Docker Hub.
        Hostname: Sets the hostname of the MySQL container to "db."
        Environment Variables:
          MYSQL_ROOT_PASSWORD: Sets the root password for MySQL.
          MYSQL_DATABASE: Creates a MySQL database named "edutech_account."
        Healthcheck: Monitors the health of the MySQL service using a ping command. 
        Ports: Maps host port 3306 to container port 3306 for MySQL connections.
        Networks: Connects to the "spin-off-backend" network.
        Volumes: Mounts a volume named db-data to persist MySQL data.

    - User API Service (userapi), API Gateway Service (apigateway), Content API Service (contentapi):
        Build: Builds the API services using the specified Dockerfile target "final."
        Restart: Restarts the services always.
        Environment Variable: Sets the ASP.NET Core environment to "Development."
        Ports: Maps host ports to container ports for each service.
        Networks: Connects to the "spin-off-backend" network.
        Depends On: Specifies that each API service depends on the MySQL service being healthy before starting.

    - MongoDB Service (mongodb):
        Image: Pulls the latest MongoDB image from Docker Hub.
        Container Name: Sets the container name to "edutech_mongodb."
        Hostname: Sets the hostname to "localhost."
        Command: Installs updates and then starts the MongoDB service.
        Environment Variable: Sets the MongoDB initialization database to "EdutechContentDB."
        Ports: Maps host port 27017 to container port 27017 for MongoDB connections.
        Volumes: Mounts a volume named mongodb_data to persist MongoDB data.
        Networks: Connects to the "spin-off-backend" network.
        Healthcheck: Monitors the health of the MongoDB service using a ping command.


    - Networks:Defines a user-defined bridge network named "spin-off-backend" to connect the services.

    - Volumes:Defines two volumes, db-data and mongodb_data, for persisting data for MySQL and MongoDB, respectively.


-  But there are a few changes we have to make before running the application,

   for example:- 
- 💡 Inside the ocelot.userapi.json file
     we have written host, port, and DownstreamScheme and we have to change that before running because we are  containerizing our database and it will not be accessible to our localhost we should name it as the service name mentioned in the compose file.

       NOTE:- Open ocelot.userapi.json
       - DownstreamScheme": "https" = DownstreamScheme": "http"
       - "Host": "localhost" = "Host": "userapi"
       - "Port": 7057 = "Port": 8080

        NOTE:- If you are using Vim or vi editor, here is a shortcut you can use,
       to change one word repeatedly you can simply use a trick,

        Follow the steps:
         - Open "ocelot.userapi.json"
         - Press Esc
         - Enter ":%s/old-word/new-word/gc" and press enter, this will need your confirmation each word is  going to replace

           - Example:- ":%s/localhost/userapi/gc"
           - Example:- ":%s/https/http/gc"
           - Example:- ":%s/7057/8080/gc"

   -  we have to do it for each block so that API can connect to the database.
   
   
   -  Open Directory User.API and inside that find appsetting.Development.json and appsetting.json
     we have to edit the Server name in both the files in line.

       - "UserDbConnection": "Server=<servicename>;User ID=<USER>root;Password=<MYSQL_ROOT_PASSWORD>;Database=<DATABASE_NAME>edutech_account"
          
           - "Server=localhost" = "Server=db" 

- Steps to run the project
  - After cloning the repo and making changes as above.

  - you can run "docker-compose up --build"

   - 🙂 Congratulations, Now you see all the containers are running.
     - You can access your services on
       - User API: http://localhost:7057
       - API Gateway: http://localhost:8080
       - Content API: http://localhost:8090
