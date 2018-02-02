# .net Core 2.0 Web API deployed to QNAP Nas using Docker container on Container Station
The purpose of this simple project is to help devs like me that likes to host dot net core application in the intranet, using QNAP Container Station.
This application is just a wrapper around docker, with a custom user interface.


## Environment setup

- Windows 10 X64 Professional
- Docker for Windows 17.06.2
- Visual Studio 2017 Enterprise 15.5.4
- QNAP Nas TS-253A 4.3.4.0435
- Container Station 1.7.26.69
- QNAP Docker Version 1.11.2


## Project Creation
- Open Visual Studio > File > New > Project
- Select the project type  in Visual C# > .NET Core > ASP.NET Core Web Application
- Choose a name and location and confirm
- Select .NET Core version 2.0, Web API, Enable Docker Support with OS: Linux and confirm
- Right click on the Web project, go to Package > Package version and set any version you might want to use, note this is necessary because by default, even if the version is visible, this is not present in the project file, but this is needed for the deployment script


## Run the project locally
At this stage you should have the docker-compose project as startup project.
If you run the application using F5, you will notice in the output window that it will compile the solution but also will build the docker image and container in the local docker for windows.

Automatically your default browser will open, and you will be able to see the output of the default hard coded invocation of a GET request on the Values controller.


## QNAP Container Station
_"Non é tutto oro quello che luccica"_

Unfortunately the Container Station uses a very old Docker Server and Client, ant the networking configuration requires some tuning.
This will require us to do three important things

1) Change the default Dockerfile generated by Visual Studio
2) Force the Docker Client API to use an older version
3) Create a custom network specifically designed for QNAP nas, in order to properly use bridged configuration, and mostly reuse the same network for all the container you might want to deploy in the future


## Preparing for the NAS deployment
I have been playing with docker since the very early days and I spent so much time fiddling with the initial docker integration, at the time to get something to run was a long painful and not always rewarding process. Microsoft was updating very often .net Core and so was Docker for Windows, causing issues everywhere. I remember when there was a time you had to copy SDK folders here and there ... but I don't think this is relevant for you.

My point is that so far you might be saying, super simple... why this guy spent time to create this project... Start playing with this stuff, then youll tell me :).

So the new wonderful Dockerfile generated buy the docker support flag, is a cutting edge version that uses Docker multi stage build feature. This feature has been introduced recently in the version 17 of the Docker Server, unfortunately in the QNAP NAS we are years behind, and the Dockerfile as it is won't work.

The original, automatically generated file is optimised and follows all the steps necessary to restore nuget packages, build, publish and define the entrypoint, but as I said, until QNAP updates the Docker server this won't work.

```
FROM microsoft/aspnetcore:2.0 AS base
WORKDIR /app
EXPOSE 80

FROM microsoft/aspnetcore-build:2.0 AS build
WORKDIR /src
COPY *.sln ./
COPY Application.Api/Application.Api.csproj Application.Api/
RUN dotnet restore
COPY . .
WORKDIR /src/Application.Api
RUN dotnet build -c Release -o /app

FROM build AS publish
RUN dotnet publish -c Release -o /app

FROM base AS final
WORKDIR /app
COPY --from=publish /app .
ENTRYPOINT ["dotnet", "Application.Api.dll"]

```

In order to be able to use the Dockerfile to create an image directly on the nas, we need to use the following version of the file.
This version only copies across the files in the publish folder.

```
FROM microsoft/aspnetcore
ARG source
WORKDIR /app
EXPOSE 80
COPY ${source:-obj/Docker/publish} .
ENTRYPOINT ["dotnet", "Api.dll"]

```

## Deployment process
To be able to deploy to the NAS, we will need to do the following operations:
- Restore nuget packages
- Build the solution
- Publish the application to a folder
- Force the Docker Client API version to use 1.23, the version (old) in the NAS
- Invoke _docker build_ specifying the Dockerfile to use and the source directory (Publish folder)
- We will need to create a new Network in the nas, for a properly working bridge configuration, for this we will use _docker network create_
- Create a new container using _docker create_ and a series of parameter


## Deployment PowerShell script
TODO :D

For the image version, the script uses the web api project version.

To generate a random mac address I use [this tool](https://justynshull.com/macgen/macgen.php).


## Execute the deployment
My PowerShell script uses [PSake](https://github.com/psake/psake) to handle the deployment process.
To install Psake open PowerShell and execute the following command:

`Install-Package psake `

Move inside the folder containing the solution file and run the following command:

`Invoke-psake .\Build\build.ps1 publish `

This command will invoke the psake task inside the build.ps1 file named publish.
This task will trigger all the necessary steps to be able to deploy.

If you want to see which other tasks are available, check the content of the build.ps1 file or run the command:

`Invoke-psake .\Build\build.ps1 `

Now browse the following url, to view the Values controller result:

`http://192.168.0.142/api/values `