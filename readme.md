## Overview

This project describes how to create an Angular application using a VsCode dev container. It showcases various techniques to improve the development experience including:

* Creating the project without having node or any other dev dependencies installed on the dev workstation.
* Debugging.
* Building and running the service using docker-compose.
* Attaching the dev container to the already running container and controlling the running process.
* Uploading the project to Github.

The requirements for this guide are:

* WSL2 installed.
* A Linux disro installed on WSL2 whose filesystem will be where the project directory lives.
* Docker Desktop installed and configured to use Linux containers.

Most Docker container base images will not have Git installed. This means that when VsCode has the project open in the dev container none of the VsCode Git integration will work. You could perform Git operations on the project from the host dev workstation WSL command line, or from a separate VsCode instance that has opened the project from its WSL directory, but that's not a pleasing experience. The most streamlined approach is to install Git in the dev container which enables the Git integration in VsCode to function as normal and that is the approach used in this guide.

## Steps to Create

#### Creating the inital project

Create a project directory on your WSL2 filesystem.

Open the project directory in VsCode

Create readme.md file

Create Dockerfile

    FROM node:lts-alpine

    RUN apk add git
    
    WORKDIR /docker-angular1

Add docker-compose.yml.

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

Generate the new Angular project in the containers workspace directory using `ng new <app-name> --directory .` After creating the Angular project I like to rename the Angular README.md file to readme-angular.md.

In `package.json` edit the `start` command to `ng serve --host 0.0.0.0`

Run `npm start`, you should now be able to access the application in a browser on the host development machine at `localhost:4200`

#### Debugging the project

In VsCode click the Debug button on the navigation sidebar.

Click the link to create a new launch configuration, add the `Chrome Attach` launch configuration.

In `launch.json` change the port on the `url` setting to `4200`.

Add a constructor with a `console.log` line to `app.component.ts`

    constructor() {
      console.log('Hello World!');
    }
    
Place a breakpoint on the `'console.log` line, press the button to start debugging.

The browser should launch and the breakpoint will be hit.

#### Enable starting the service using docker-compose.

Currently the application can be run because we manually ran `npm install` when setting up the dev constainer and then ran `npm start`. The following steps are necessary to enable the service to be rebuilt and run using docker-compose.

Add the following to the end of the `Dockerfile`

    COPY ["package.json", "package-lock.json", "./"]
    RUN npm install
    COPY . .

Add the following to the end of the service definition in `docker-compose.yml`

    command: npm start

To test this close the project in VsCode (or reopen in WSL). In a separate terminal run `docker ps` to verify that the dev container has exited. In a browser navigate to `localhost:4200`, it should not be reachable.

In the same separate terminal navigate to the project directory and run `docker-compose up -d --build`. This will rebuild the container from scratch, copy the `package.json` across, install all the dependencies and run the service (this may take some time to complete). Once this has completed you will be able to navigate to `localhost:4200` and see the hosted application.

Note that since we manually installed `ng` globally now that we have rebuilt the container the global installation of `ng` is no longer present. This isn't a problem though since the `ng` cli is also installed locally when the project is created and can be invoked using `npm run ng` (arguments to `ng` must be separated from the call to `npm` using -- e.g. `npm run ng -- serve --help)`. Alternatively `ng` can be invoked from the `node_modules/.bin` directory.

With the container running it is possible to make changes to the project in VsCode opened on the WSL directory and on saving they will be recompiled and the browser reloaded. If your WSL user id is different to the root user in the dev container then VsCode will signal an error on attempting to save a modified file. This can be resolved by running `sudo chown <wsl_username> <filename>`. An better solution to the permission issue would be to run the dev container with a user having the same id as your WSL user, but there are currently issues with this approach when using bind mounted volumes. At any rate, modifying the code while the container is running via docker-compose isn't a particularly useful feature, but this demonstrates that it is nevertheless possible.

#### Improve the experience connecting VsCode to an already running container

With the service run via `docker-compose up -d` opening the dev container in VsCode will cause VsCode to attach to the already running container. This is nice but the usefulness of this is limited since the terminal spawned by opening the VsCode dev container will not attach to the already running service process in the container. So if your container is already running `ng serve` when you attach to it you have no way to terminate this process and execute other commands instead like `npm install` or `npm test` or the restart the `ng serve`, and even if you could terminate the running `ng serve` that would not be useful since that would cause the container to exit anyway since that is the process that is keeping the container alive.

There is a solution to this that involves the use of `tmux`. `tmux` is a terminal multiplexer, which among other things enables processes to be spawned in one terminal and then attached to and controlled from another terminal.

First install `tmux` in the container by adding the following to the Dockerfile after the line which installs `git`. Note that the shell initialisation script that this creates is designed to work specifically for alpine Linux containers which use `ash` as the default shell. A different initialisation may be required for non-alpine Linux containers using `bash`. 

    # Install tmux
    RUN apk add tmux
    
    # Create a shell initialisation script that checks whether the current shell is running
    # within a tmux session, and if not attaches to an existing session.
    ENV ENV="/root/.initsh"
    RUN echo "if [ \"$TMUX\" = \"\" ]; then" > "$ENV"
    RUN echo "tmux attach -t my_session" >> "$ENV"
    RUN echo "fi" >> "$ENV"

Then replace the `command` in `docker-compose.yml` with

    command: sh -c "tmux new -d -s my_session;
      tmux send-keys -t my_session npm Space start C-m;
      tmux attach -t my_session"

Then rebuild the dev container, exit from the dev container and launch the service from a terminal using `docker-compose up -d` and verify the application is running by opening it in a browser at `localhost:4200`

Open the project dev container in VsCode. You will find the `ng serve` process running in the integrated terminal in a `tmux` session. You can update the code and your changes will update in the running app, you can also exit the running `ng serve` process using `^C` and run other `npm` commands like `npm install` or `npm test` and restart the service again using `npm start`.

#### Upload the project to Github

Since git was installed in the container to start with, the project will have been created with a Git repository already initialised. So all that is required to upload the project to Github is to commit the changes to any modified files, and then in the VsCode status bar click the `Publish to Github` button. Assuming you have a Github account, you will be prompted to select either a public or provide Github repository and the project will be uploaded.

## Conclusion

This process to construct the project takes a somewhat iterative approach, adding and testing functionality a bit at a time. There is a fair amount of complexity here and building the project up in this manner and testing at each stage affords the best chance of tracking down any issues if something doesn't work. Once familiarity with the tools and configuration has been gained a project with this configuration can be more quickly constructed in one go.