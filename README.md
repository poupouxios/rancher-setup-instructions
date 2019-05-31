#Running Rancher 2.x and deploy an image

This readme will take you step by step to setup Rancher and how you can orchestrate a web application with a database to communicate and work.

Before running Rancher 2.x make sure you stop anything that reserves ports 80 and 443 eg. nginx,apache2 etc, as Kubernetes Ingress nginx (load balancer) will try to reserve the port 80 and if its already occupied it will not start.

## Setup Cluster and Node ready for deployment

- Run the below command to start Rancher UI
  - `docker run -d --restart=unless-stopped -v /opt/rancher:/var/lib/rancher -p 8080:80 -p 8443:443  rancher/rancher:latest`
  - The (-v /opt/rancher:/var/lib/rancher) as Rancher mentions `it uses etcd as datastore. When using the Single Node Install, the embedded etcd is being used. The persistent data is at the following path in the container: /var/lib/rancher. You can bind mount a host volume to this location to preserve data on the host it is running on. When using RancherOS, please check what persistent storage directories you can use to store the data.`

- If you are running it on Linux do a `docker inspect <container-name-of-rancher-running>` and go towards the end to find the IP Address. Or you can use localhost:8080 for more convenience.

- Set a password for your Rancher along with the URL to access it.

- Next step is to create a custom cluster. Click Add Cluster.

- As you will see, Rancher offers you the ability to create a cluster in various hosting providers. For the current setup, we will select custom to run it locally.

- For this example, we will name our cluster 'MainCluster'. We will use this name in the below steps.

- Select on Network Provider the Calico option (it seems to work fine for me this). More info about Network options can be found here https://rancher.com/docs/rke/latest/en/config-options/add-ons/network-plugins/

- Click Next. This will save the Cluster and will show you a screen to create a Node. Node is the server where the actual pods/containers will run.

- For your Master Node, make sure to select the etcd and control pane. For any other Nodes you add, you can leave only the worker ticked.

- Run the command shown on that page to the Node in order to bind it to Rancher.

- When you run the command, you can monitor the log by doing `docker log -f <container-name-of-rancher-ui>`

- Wait until you see the Cluster and Node are Active.

- You can see on Rancher the progress of the Node.

- When the Node is ready, you can click on it and you will see  CPU and Memory monitoring and how many pods it runs.


##Orchestrate containers

- Now that we have a Node running, lets setup some containers to run.

- On Navigation Menu, click Projects/Namespaces. You will see there are 2 Namespaces - one Default and one System. The System one has all the necessary Kubernetes services to begin orchestration. The Default Namespace is to set our pods/containers to run. You can always create your own Namespace and add there your pods.

- Click on `Project: Default` to get inside.

- You will see now some new sections - Workloads, Load Balancing, Service Discovery, Volumes, Pipelines. We will use some of them, as we need to make some configurations to make the orchestration work.

- Before we go and set anything, we have to define where to pull images. On Navigation Menu, select Resources -> Registries and click Add Registry.

- You will see 3 options (Docker hub, Quay.io and Custom if you have your own Registry). For our example, we are going to use Docker Hub. Specify your username and password on right and press Save.

- Now we are ready to pull some docker images.

- Click on Workloads and click Deploy.

- On Workloads we set a unique name for each image we are going to run along with setting any environment variables the application depends, mount any volumes we want to share code or files etc.

- I am going to use one of my docker images called "poupou/lumen-swoole:swoole_0.3" which sets up Lumen Framework with Swoole. If you are curious what is this, you can find more here https://github.com/swooletw/laravel-swoole

- Fill up the below:
  - Name: swoole

  - Docker Image: poupou/lumen-swoole:swoole_0.3

  - Add Port:
    - Publish the container port: 80
    - As a: Cluster IP (internal only)

  - Environent Variables: (Click Add Variable and paste the below. Rancher will populate correctly)
    APP_NAME=Swoole  
    APP_ENV=production  
    APP_KEY=base64:cLvcXoMvzPQ1r5Czjir8fRQ4qwzOYXp2m+AhtJwRsG4=  
    APP_DEBUG=true  
    APP_LOG_LEVEL=debug  
    DB_CONNECTION=mysql  
    DB_HOST=swoole-mysql  
    DB_PORT=3306  
    DB_DATABASE=lumendb  
    DB_USERNAME=lumenuser  
    DB_PASSWORD=test1234!  
    SWOOLE_HTTP_DAEMONIZE=true  
    SWOOLE_HTTP_REACTOR_NUM=4  
    SWOOLE_HTTP_WORKER_NUM=4  
    SWOOLE_HTTP_TASK_WORKER_NUM=4  
    SWOOLE_HOT_RELOAD_ENABLE=true  

  - Volumes (Select Bind-mount a directory from the node)
    - Volume name: swoole-lumen-storage
    - Path on the node: (set a path that Rancher can access on your node eg. /home/valentinos/swoole/storage)
    - The Path on the Node must be: A directory, or create if it does not exist
    - Mount Point: /var/www/storage

- After filling the above, press Launch and wait until the Workload becomes Active. If something doesn't work, click on the Workload and inside it you will see the number of Pods it spin up. You can click on the three dots on the right of the pod that is trying to run and select View logs. This will show you the container logs and see if something went wrong.

- Now that we setup a Workload, we need a database storage to connect it with the web app.

- Click to Deploy a new Workload and fill up the below things:
  - Name: swoole-mysql

  - Docker image: mysql:5.7

  - Environment Variables: (Click Add Variable and paste the below. Rancher will populate correctly)
    MYSQL_USER=lumenuser  
    MYSQL_PASSWORD=test1234!  
    MYSQL_DATABASE=lumendb  
    MYSQL_ROOT_PASSWORD=veryS3cur3P4ssw0rd1234!  

  - Volumes (Select Bind-mount a directory from the node)
    - Volume name: swoole-mysql-db
    - Path on the node: (set a path that Rancher can access on your node eg. /home/valentinos/swoole/db)
    - The Path on the Node must be: A directory, or create if it does not exist
    - Mount Point: /var/lib/mysql

- If you have saw above, we set the name of the Workload to be swoole-mysql. The same name is set inside Service Discovery. This name will act as host to communicate with the database through the Swoole app. You can see it on DB_HOST Environment variable value above.

- Now that we set up our Workloads, its time to orchestrate them to work. Select now the Load Balancing and press Add Ingress.

- This is what publishes a Workload, sets it live and allows access from the outside world to the pods/containers.

- Fill up the below parameters:

  - Name: swoole-website

  - Specify a hostname to use (we are going to set a custom hostname bounded to the xip.io as we don't have a domain available. If you have a domain, set the exact domain name eg. google.com): swoole.<server-ip>.xip.io (for more information about to setup a subdomain of xip.io visit the site xip.io)

  - Target Backend: Press the Service and remove the Workload row from below. Select Target Swoole and Port 80.

- Press Save and wait until the Initializing becomes Active.

- If everything goes well, visit `http://swoole.<server-ip>.xip.io` and you will see Lumen load on the left showing `Lumen (5.7.7) (Laravel Components 5.7.*)`

- Now the orchestration happens and the `swoole` workload communicates with the `swoole-mysql` workload.


##Access container logs and shell

- Let's say we run this application for the first time and we want to run migrations to the database. Somehow we need access on the codebase to run the migrations.

- This can be done inside a container. Rancher offers a shell view that gives you access inside the container and run any available commands that your image offer.

- Click on Workloads and click on `swoole`. Find one pod that is running and click the three dots on right.

- Click Execute Shell. This will bring a modal terminal window and will connect you inside the container.

- Now you can run any commands you want inside your container. If you have more than one container, you don't have to execute the same command on all of them as for our example the migrations will be done on database, which all the containers have access.

- For our example, navigate to `cd /var/www` and execute `php artisan migrate`. Accept to run this on production. If everything goes well you should see a message:

  `Migration table created successfully.
  Nothing to migrate.`

- Now lets make sure that the table migrations is being created in the database. Visit back the Workload and select `swoole-mysql` and click the three dots of the running pod and select Execute Shell.

- Execute now the command `mysql -u lumenuser -p lumendb` and enter the password from above.

- If you do `show tables;` you will see the migrations table there, which indicates that the communication from `swoole` pod to `swoole-mysql` worked.

##Extra steps for more familiarization

Now you are ready to orchestrate more applications. You can try now and use the `mcr.microsoft.com/dotnet/core/samples:aspnetapp` image to setup a workload for this and see how an ASP.NET core application runs through container.

##References

- https://rancher.com/docs/rancher/v2.x/en/installation/single-node/
- https://octoperf.com/blog/2018/06/04/rancher-2-getting-started/
- https://computingforgeeks.com/how-to-install-and-use-rancher-2-to-manage-containers-on-ubuntu-18-04-lts/
