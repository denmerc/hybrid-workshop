# Deploying Multi-OS applications to Docker Swarm
With the release of Docker overlay networking for Windows Server 2016, it's not possible to create swarm clusters that include Windows Servers. This could be an all Windows cluster, or a hybrid cluster of Linux and Windows machines. 

In this lab we'll build a hybrid cluster, and then deploy both a Linux and Windows web app, as well as an application that includes both Windows and Linux components. 	

> **Difficulty**: Intermediate (assumes basic familiarity with Docker) 

> **Time**: Approximately 60 minutes

> **Tasks**:
>
> * [Prerequisites](#prerequisites)
> * [Task 1: Build a Hybrid Swarm](#task1)
>   * [Task 1.1: Create the Swarm Manager](#task1.1)
>   * [Task 1.2: Add a Worker Node](#task1.2)
>   * [Task 1.3: Examine the Cluster](#task1.3)
> * [Task 2: Deploy a Linux Web App Service](#task2)
>   * [Task 2.1: Clone the Demo Repo](#task2.1)
>   * [Task 2.2: Build and Push Your Image to Docker Hub](#task2.2)
>   * [Task 2.3: Deploy the Web App](#task2.3)
> * [Task 3: Deploy a Windows Web App Service](#task2)
>   * [Task 3.1: Clone the Demo Repo](#task2.1)
>   * [Task 3.2: Build and Push Your Image to Docker Hub](#task2.2)
>   * [Task 3.3: Deploy the Web App](#task2.3)
> * [Task 4: Deploy a Multi-OS Application](#task4)
>   * [Task 4.1: Examine the Docker Compose File](#task4.1)
>   * [Task 4.2: Deploy the Application](#task4.2)
>   * [Task 4.3: Verify the Running Application](#task4.3)

## Document conventions

When you encounter a phrase in between `<` and `>`  you are meant to substitute in a different value. 

For instance if you see `$ip = <ip-address>` you would actually type something like `$ip = '10.0.0.4'`

## <a name="prerequisites"></a>Prerequisites

You will be provided a set of  virtual machines (one Windows and one Linux) running in Azure, which are already configured with Docker and some base images. You do not need Docker running on your laptop, but you will need a Remote Desktop client to connect to the Windows VM, and an SSH client to connect into the Linux one. 

### RDP Client
- Windows - use the built-in Remote Desktop Connection app.
- Mac - install [Microsoft Remote Desktop](https://itunes.apple.com/us/app/microsoft-remote-desktop/id715768417?mt=12) from the app store.
- Linux - install [Remmina](http://www.remmina.org/wp/), or any RDP client you prefer.

### SSH Client
- Windows - [Download Putty](http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html)
- Linux - Use the built in SSH client
- Mac - Use the built in SSH client

> **Note**: When you connect to the Windows VM, if you are prompted to run Windows Update, you should cancel out. The labs have been tested with the existing VM state and any changes may cause problems.

### Docker ID
You will build images and push them to Docker Hub, so you can pull them on different Docker hosts. You will need a Docker ID.

- Sign up for a free Docker ID on [Docker Hub](https://hub.docker.com)

## <a name="task1"></a>Task 1: Build a Hybrid Swarm

Our first step will be to create a two node swarm cluster. We'll make the Linux node the manager node, and our Windows node will be the worker node. 

> **Note**: Window Server 2016 machiens can also be manager nodes.

### <a name="task1.1"></a>Task 1.1: Create the Swarm Manager

Either in a terminal window (Mac or Linux) or using Putty (Windows) SSH into your Linux node. The DNS name, username and password should have been provided to you.

```
ssh docker@<linux node DNS name>.westus2.cloudapp.azure.com
```

Create your swarm by issuing the `docker swarm init` command. This command will create a swarm cluster, and add the current node as a manager.

```
$ docker swarm init
Swarm initialized: current node (z42u37g25lrmcgbpyef9fd06r) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-4qm2iur0lkqjmmxlfivyj7rdn9nsso216vaxybhojgmbwa3su7-3vzae67xszr1yphz6flr9emff \
    10.0.2.32:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

Copy the join command and paste it into a text editor, then remove the `\` and make the command a single line.

It should end up looking something like this:

```
docker swarm join --token SWMTKN-1-4qm2iur0lkqjmmxlfivyj7rdn9nsso216vaxybhojgmbwa3su7-3vzae67xszr1yphz6flr9emff 10.0.2.32:2377
```

### <a name="task1.2"></a>Task 1.2: Add a Worker Node

Use your RDP client to connect into your Windows node. The DNS name, username, and password should have been provided to you. 

Once connected to your Windows Server 2016 VM open a Powershell window by clicking the `Start` button (which looks like a flag in the lower left corner) and then click the Windows Power Shell icon (Do NOT choose Windows Powershell ISE) near the top right.

In the Powershell window paste in your Docker swarm join command. This command will contact the swarm manager you just created and use the token supplied to join the Swarm cluster.

```
docker swarm join --token SWMTKN-1-4qm2iur0lkqjmmxlfivyj7rdn9nsso216vaxybhojgmbwa3su7-3vzae67xszr1yphz6flr9emff 10.0.2.32:2377
This node joined a swarm as a worker.
```
> **Note**: Sometimes joining the swarm will cause your RDP connection to reset, if this happens simply reconnect

### <a name="task1.3"></a>Task 1.3: Examine the Cluster

Switch back to your Linux VM.

To see the nodes in your cluster issue the `docker node ls` command.

```
$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
phud37l5prpy4bk76pfdkfofw     win-pdx-20          Ready               Active
z42u37g25lrmcgbpyef9fd06r *   lin-pdx-20          Ready               Active              Leader
```

We can verify the operating system family for each node by using docker inspect.

> **Note** The command below uses the hostnames show above, your hostnames will be different

```
$ docker node inspect win-pdx-20 | grep OS
                "OS": "windows"
$ docker node inspect lin-pdx-20 | grep OS
                "OS": "linux"
```
Let's look at the OS information in a bit more detail. Issue the `docker node inspect` command on your Linux node.

`$ docker inspect lin-pdx-20`

Scroll up in your terminal until you find the `Description` section. It should look similar to this:

```
"Description": {
            "Hostname": "lin-pdx-20",
            "Platform": {
                "Architecture": "x86_64",
                "OS": "linux"
            },
```

What we want to notice here is the hierarchy. We inspected the node, and the `OS` is listed under `Platform` so the full path for the `OS` label is `node.platform.OS`. This information will be used in a later part of the lab.

## <a name="task2"></a>Task 2: Deploy a Linux Web App

Now that we've build our cluster, let's deploy a couple of web apps. These are simple web pages that allow you to send a tweet. One is built on Linux using NGINX and the other is build on Windows Server 2016 using IIS.  

We're going to clone the workshop repo onto each machine, and then build a the webapp for each operating system.

Let's start on the Linux node.

### <a name="task2.1"></a> Task 2.1: Clone the Demo Repo

Make sure you're in your home directory on your Linux VM

`$ cd ~`

And then clone the repository.

```
$ git clone https://github.com/mikegcoleman/hybrid-workshop.git
Cloning into 'hybrid-workshop'...
remote: Counting objects: 13, done.
remote: Compressing objects: 100% (10/10), done.
remote: Total 13 (delta 1), reused 10 (delta 1), pack-reused 0
Unpacking objects: 100% (13/13), done.
Checking connectivity... done.
```

You now have the necessary demo code on your Linux VM. 

### <a name="task2.2"></a> Task 2.2: Build and Push the Linux Web App Image

Change into the `linux_tweet_app` directory.

`$ cd ~/hybrid-workshop/linux_tweet_app/`

#### Build the Docker image

Next use `docker build` to build your Linux tweet web app Docker image.

> **Note**: Be sure to user your own Docker ID when you build the image.

> **Note**: Feel free to examine the Dockerfile in this directory if you'd like to see how the image is being built.

```
$ docker build -t <your docker id>/linux_tweet_app .
Sending build context to Docker daemon  4.096kB
Step 1/4 : FROM nginx:latest
latest: Pulling from library/nginx
ff3d52d8f55f: Pull complete
b05436c68d6a: Pull complete
961dd3f5d836: Pull complete
Digest: sha256:12d30ce421ad530494d588f87b2328ddc3cae666e77ea1ae5ac3a6661e52cde6
Status: Downloaded newer image for nginx:latest
 ---> 3448f27c273f
Step 2/4 : COPY index.html /usr/share/nginx/html
 ---> 72d22997a765
Removing intermediate container e262b9220942
Step 3/4 : EXPOSE 80 443
 ---> Running in 54e4ff1b39a6
 ---> 2b5bd87894cd
Removing intermediate container 54e4ff1b39a6
Step 4/4 : CMD nginx -g daemon off;
 ---> Running in 54020cdec942
 ---> ed5f550fc339
Removing intermediate container 54020cdec942
Successfully built ed5f550fc339
Successfully tagged <your docker id>/linux_tweet_app:latest
```
Next You'll log into Docker hub and push our image up there for future use.

#### Push the image to Docker Hub

```
$ docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: <your docker id>
Password: <your docker id password>
Login Succeeded
```
After you log in, use `docker push` to upload your image up to Docker Hub.
```
$ docker push <your docker id>/linux_tweet_app
The push refers to a repository [docker.io/<your docker id>/linux_tweet_app]
4d8d2fc9cf5a: Pushed
08e6bf75740d: Mounted from library/nginx
f12c15fc56f1: Mounted from library/nginx
8781ec54ba04: Mounted from library/nginx
latest: digest: sha256:c89bbabda050e5eba0f8ee3bf40a0c672da08107e612e58062b97988ea463277 size: 1155
```

### <a name="task2.3"></a> Task 2.3: Deploy the Web App

Now let's run our application by by creating a new service. 

Services are application building blocks (although in many cases an application will only have one service, such as this example). Services are based on a single Docker image. Tasks are the individual Docker containers that execute the application. When you create a new service you instantiate at least one task automatically, but you can scale the number of tasks up to meet the needs of your service. 

#### Create a new service

To create a new service you issue the `docker service create` command. 

```
$ docker service create \
   --detach=true \
   -p 8080:80 \
   --name linux_tweet_app \
   --constraint 'node.platform.os == linux' \
   <your docker id>/linux_tweet_app
sbkz4dl6slrd6xl7e6rp1qlt2
```
Let's look at each part of that command:

- `docker service create`: Creates a new service based on the supplied image (`<your docker id>/linux_tweet_app`)

- `--detach=true`: Runs our service in the background

- `-p 8080:80`: Instructs Docker to route any requests coming in to our host VM on port 8080 to our service running on port 80.

- `--name linux_tweet_app`: Applies a name to our service (if this is omitted Docker will choose one at random)

- `--constraint 'node.platform.os == linux'`: This tell swarm to only start this service on a Linux-based host

	> **Note**: This is the label we looked at earlier when we did the `docker node inspect` command

#### Verify our service and tasks started

We can use the `docker service ls` command to verify our service was created.

```
$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                                 PORTS
sbkz4dl6slrd        linux_tweet_app     replicated          1/1                 mikegcoleman/linux_tweet_app:latest   *:8080->80/tcp
```

> **Note**: When the service is fully deployed it should read `1/1` under `REPLICAS`

And we can check the tasks in our service by running `docker service ps`.

```
$ docker service ps linux_tweet_app
ID                  NAME                IMAGE                                 NODE                DESIRED STATE       CURRENT STATE                ERROR               PORTS
z8fbzmkd92kv        linux_tweet_app.1   mikegcoleman/linux_tweet_app:latest   lin-pdx-02          Running             Running about a minute ago
```
#### Check the running web app
Finally, in your web browser navigate to `http://<your linux node dns name>:8080`

You should see your tweet application running. Feel free to send a tweet, since it's running in container your credentials are not saved.

## <a name="task3"></a>Task 3: Deploy the Windows version of our Twitter web application

Now we'll deploy the Windows version of the twee app.

### <a name="task3.1"></a> Task 3.1: Clone the Demo Repo
Move back to your Windows Server 2016 virtual machine, and open a PowerShell window.

Let's create a new directory, move into it and then clone the repo:

`PS C:\Users\docker> mkdir scm`

`PS C:\Users\docker> cd .\scm`

```
C:\Users\docker\scm> git clone https://github.com/mikegcoleman/hybrid-workshop.git
Cloning into 'hybrid-workshop'...
remote: Counting objects: 13, done.
remote: Compressing objects: 100% (10/10), done.
remote: Total 13 (delta 1), reused 10 (delta 1), pack-reused 0
Unpacking objects: 100% (13/13), done.
```

### <a name="task3.2"></a> Task 3.2: Build and Push the Windows Web App Image

Now let's move into the tweet app directory, buld the image, and push it to Docker Hub. 

#### Build the Windows image

Once again use `docker build` to build a new image.

> **Note**: Be sure to use your docker id when tagging the docker image

```
PS C:\Users\docker\scm> cd .\hybrid-workshop\windows_tweet_app\


PS C:\Users\docker\scm\hybrid-workshop\windows_tweet_app\docker build -t <your docker id>/windows_tweet_app .
Sending build context to Docker daemon  6.144kB
Step 1/10 : FROM microsoft/windowsservercore
 ---> 590c0c2590e4

<output snipped>

Step 10/10 : HEALTHCHECK CMD powershell -command     try {      $response = Invoke-WebRequest http://localhost -UseBasic
Parsing;      if ($response.StatusCode -eq 200) { return 0}      else {return 1};     } catch { return 1 }
 ---> Running in ab4dfee81c7e
 ---> d74eead7f408
Removing intermediate container ab4dfee81c7e
Successfully built d74eead7f408
Successfully tagged <your docker id>/windows_tweet_app:latest
```
> **Note**: It will take a few minutes for your image to build

#### Push the Windows Image

Let's change back to our home directory and log into Docker Hub so we can push your image up to your repository.

```
PS C:\Users\docker\scm\hybrid-workshop\windows_tweet_app> cd ~

PS C:\Users\docker> docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: <your docker id>
Password:
Login Succeeded
```
Once we are logged in we can push your new image up to Docker Hub.

```
PS C:\Users\docker> docker push <your docker id>/windows_tweet_app
The push refers to a repository [docker.io/mikegcoleman/windows_tweet_app]
5d08bc106d91: Pushed
74b0331584ac: Pushed
e95704c2f7ac: Pushed
669bd07a2ae7: Pushed
d9e5b60d8a47: Pushed
8981bfcdaa9c: Pushed
25bdce4d7407: Pushed
df83d4285da0: Pushed
853ea7cd76fb: Pushed
55cc5c7b4783: Skipped foreign layer
f358be10862c: Skipped foreign layer
latest: digest: sha256:e28b556b138e3d407d75122611710d5f53f3df2d2ad4a134dcf7782eb381fa3f size: 2825
```

### <a name="task3.3"></a> Task 3.3: Deploy the Web App

We're going to run your application as a service on our swarm cluster. Because of this we need to issue the `docker service create` command from your manager node. With swarm, any work with the swarm needs to be done from a manager node.

> **Note**: If you want to see what happens when you try to issue swarm commands from a worker simply issue the 'docker node ls' command from your Windows Server 2016 vm. You should see an error indicating that the command can only be run from a manager node.

#### Create the Windows tweet app service

Head back over to your Linux VM (which is our swarm manager) and deploy our Windows tweet app.

```
$ docker service create \
   --name windows_tweet_app \
   --publish mode=host,target=80,published=8081 \
   --detach=true \
   --name windows_tweet_app \
   --constraint 'node.platform.os == windows' \
   <your docker id>/windows_tweet_app
```
> **Note**: In this case we set the constraint to 'node.platform.os == windows' to ensure the service is only started on a Windows-based host.

> **Note**: You'll notice the format for publishing the network ports are different with our Windows application. That's because that this time Windows does not support the integrated ingress load balancing, and we need to expose the ports in "host mode". See the Docker documentation for more information on host mode.  

#### Check the service

Use `docker service ls` to see if your service is running:

```
ID                  NAME                MODE                REPLICAS            IMAGE                                   PORTS
2cax4875jqnp        linux_tweet_app     replicated          1/1                 mikegcoleman/linux_tweet_app:latest     *:8080->80/tcp
mo1y1hz592jo        windows_tweet_app   replicated          1/1                 mikegcoleman/windows_tweet_app:latest
```

>**Note**: It may take a minute for the service to start. Until it does you will see `0/1` under `REPLICAS`

You can verify the task started on your Windows node by issuing the `docker service ps` command and checking that your Windows host is listed under `NODE`

```
$ docker service ps windows_tweet_app
ID                  NAME                  IMAGE                                   NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
iw4pj3pb1b2g        windows_tweet_app.1   mikegcoleman/windows_tweet_app:latest   win-pdx-20          Running             Started 2 seconds ago
```

#### Verify the service is running

Finally, let's visit the web app by pointing our web browser to `http:<windows vm dns name>:8081`

> **Note**: Be sure to specify port *8081*

## <a name="task4"></a> Deploying a Multi-OS Application

For our last exercise we'll use a docker compose file to deploy an application that uses a Java front end designed to be deployed on Linux, with a Microsoft SQL Server back end running on windows.

### <a name="task4.1"></a> Task 4.1: Examine the Docker Compose file

Change into the `hybrid-workshop` directory

`$ cd ~/hybrid-workshop`

We'll use a Docker Compose file to instantiate our application. With this file we can define all our services and their parameters, as well as other Docker primatives such as networks. 

Let's look at the Docker Compose file:

```
version: "3.2"

services:

  database:
    image: sixeyed/atsea-db:mssql
    ports:
      - mode: host
        target: 1433
        published: 1433
    networks:
     - atsea
    deploy:
      endpoint_mode: dnsrr
      placement:
        constraints:
          - 'node.platform.os == windows'

  appserver:
    image: sixeyed/atsea-app:mssql
    ports:
      - mode: host
        target: 8080
        published: 80
    networks:
      - atsea
    deploy:
      endpoint_mode: dnsrr
      placement:
        constraints:
          - 'node.platform.os == linux'

networks:
  atsea:
```

There are two services. `appserver` is our web frontend written in Java, and `database` is our Microsoft SQL Server database. The rest of the commands should look familiar as they are very close to what we used when we deployed our tweet services manually.

One thing that is new is the creation of an overlay network (`atsea`). Overlay networks allow containers running on different hosts to communicate over a private software-defined network. In this case, the web frontend on our Linux host will use the `atsea` network to communicate with the database.

You may have used Docker Compose before to deploy multi-service applications, but with swarm we use a slightly different command: `docker stack`.

### <a name="task4.2"></a> Task 4.2: Deploy the Application

To deploy a new stack we use `docker stack create` and supply a link to our Docker Compose file as well as a name for our service (`atsea` in this case)

```
$ docker stack deploy -c docker-compose.yaml atsea
Creating network atsea_atsea
Creating service atsea_database
Creating service atsea_appserver
```

The output shows the creation of the two services, and our network.

#### Check the Status of the Application

Using `docker stack ps` will show the state of our services

```
$ docker stack ps atsea
ID                  NAME                IMAGE                     NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
dflu6i3po6ur        atsea_appserver.1   sixeyed/atsea-app:mssql   lin-pdx-21          Running             Running 7 minutes ago                       *:80->8080/tcp
wgaf4vxptafj        atsea_database.1    sixeyed/atsea-db:mssql    win-pdx-21          Running             Running 6 minutes ago                       *:1433->1433/tcp
```

You can see from the output above the two services were deployed to the two different hosts, and are now up and Running

> **Note**: It can take a few minutes for all services to start. Just keep running the `docker stack ps` command until you see both services with a `DESIRED STATE` of `Running`

### <a name="task4.3"></a> Task 4.3: Verify the Running Application

To see our running web site (an art store) visit `http://<your linux dns name>`.

This concludes our workshop, thanks for attending. 
