## Overview

This project describes how to create an Angular application using a VsCode dev container. It showcases various techniques including:

* Creating the project without having node on the dev workstation.
* Uploading to Github.
* Debugging.
* Launching the container using docker-compose then attaching the dev container and controlling the running process.

The dev container will most likely not have Git installed. This means that when VsCode has the project open in the dev container none of the VsCode Git integration will work. You could perform Git operations on the project from the host dev workstation WSL command line, or from a separate VsCode instance that has opened the project from its WSL directory, but that's not a pleasing experience. The most streamlined approach is to install Git in the dev container.

## Steps to Create

Create a project directory on your WSL2 filesystem.

Open the project directory in VsCode

Create readme file

Create Dockerfile

    FROM node:lts-alpine

    RUN apk add git
    
    WORKDIR /docker-angular1

Add docker-compose.yml. (`ng new` appears to use the service name for the node project name).

    version: "3.8"
    services:  
      docker-angular1:
        build: ./
        tty: true
        stdin_open: true
        ports:
          - 4200:4200
        volumes:
          - ./:/docker-angular1
          - /node_modules
    
Create .devcontainer.json

    {
       "dockerComposeFile": "./docker-compose.yml",
       "service": "docker-angular1",
       "workspaceFolder": "/docker-angular1"
    }

Reopen the project in a VsCode dev container by running the VsCode command `Remote-Containers: Reopen in Container`

Install the Angular cli tool globally in the dev container using `npm install -g @angular/cli` in the VsCode terminal.

Generate the new Angular project in the containers workspace directory using `ng new <app-name> --directory .`

In `package.json` edit the `start` command to `ng serve --host 0.0.0.0`

Run `np start`, you should now be able to access the application in a browser on the host development machine at `localhost:4200`


## Upload to Github

The dev container will most likely not have Git installed. This means that when VsCode has the project open in the dev container none of the VsCode Git integration will work. You could perform Git operations on you project from the host dev workstation WSL command line or from a separate VsCode instance that has opened the project from its WSL directory but that's not a pleasing experience. The most streamlined approach is to install Git in the dev container.

Update the Dockerfile to:

    FROM node:lts-alpine
    
    RUN apk add git
    
    WORKDIR /app
    
...and then run `Remote-Containers: Rebuild Container`, you can then perform Git operations from within your dev container in the usual manner.


## Next Steps

Test debugging

Add tmux support
