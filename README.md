
[Source](https://docs.docker.com/compose/django/ "Permalink to Quickstart: Compose and Django | Docker Documentation")

# Quickstart: Compose and Django inside a Vagrant Machine

Estimated reading time:  7 minutes 

This quick-start guide demonstrates how to use Docker Compose to set up and run a simple Django/PostgreSQL app hosted in a Vagrant machine. Before starting, install the Vagrant machine by using a `Vagrantfile` that will be set up to install Docker automatically, make sure you have a virtual machine provider available for Vagrant.

### Define the project components

For this project, you need to create a Vagrantfile, Dockerfile, a Python dependencies file, and a `docker-compose.yml` file. (You can use either a `.yml` or `.yaml` extension for this file.)

1. Create an empty project directory.

    You can name the directory something easy for you to remember. This directory is the context for your application image. The directory should only contain resources to build that image.

2. Create a new file called 'Vagrantfile' in your project directory.

    The Vagrantfile describes the type of machine required for a project, and how to configure and provision these machines. For more information on `Vagrantfile`, see the [Vagrant user guide][1].

3. Add the following content to the `Vagrantfile`.

    ```vagrantfile
    # -*- mode: ruby -*-
    # vi: set ft=ruby :
    
    # All Vagrant configuration is done below. The "2" in Vagrant.configure
    # configures the configuration version (we support older styles for
    # backwards compatibility). Please don't change it unless you know what
    # you're doing.
    Vagrant.configure("2") do |config|
      # The most common configuration options are documented and commented below.
      # For a complete reference, please see the online documentation at
      # https://docs.vagrantup.com.
    
      # Every Vagrant development environment requires a box. You can search for
      # boxes at https://vagrantcloud.com/search.
      config.vm.box = "ubuntu/bionic64"
      config.vm.box_version = "~> 20190314.0.0"
    
      config.vm.network "forwarded_port", guest: 8000, host: 8000
      config.vm.provision "shell", inline: <<-SHELL
        systemctl disable apt-daily.service
        systemctl disable apt-daily.timer
    
        sudo apt update
        sudo apt-get --assume-yes install apt-transport-https ca-certificates curl software-properties-common
        clear
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
        sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
        sudo apt update
        sudo apt-cache policy docker-ce
        clear
        sudo apt-get --assume-yes install docker-ce
        clear
        sudo systemctl status docker
        docker info
        sudo curl -L https://github.com/docker/compose/releases/download/1.25.1-rc1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
        sudo chmod +x /usr/local/bin/docker-compose
        docker-compose --version
      SHELL
    end
   ```
 
    The `Vagrantfile` will build your virtual machine and install Docker in it by automatically installing necessary packages.

4. Save and close the `Vagrantfile`.

5. Create a new file called `Dockerfile` in your project directory.

    The Dockerfile defines an application's image content via one or more build commands that configure that image. Once built, you can run the image in a container. For more information on `Dockerfile`, see the [Docker user guide][2] and the [Dockerfile reference][3].

6. Add the following content to the `Dockerfile`.
  
    ```dockerfile
    FROM python:3
    ENV PYTHONUNBUFFERED 1
    RUN mkdir /code
    WORKDIR /code
    COPY requirements.txt /code/
    RUN pip install -r requirements.txt
    COPY . /code/
    ```

    This `Dockerfile` starts with a [Python 3 parent image][4]. The parent image is modified by adding a new `code` directory. The parent image is further modified by installing the Python requirements defined in the `requirements.txt` file.

7. Save and close the `Dockerfile`.

8. Create a `requirements.txt` in your project directory.

    This file is used by the `RUN pip install -r requirements.txt` command in your `Dockerfile`.

9. Add the required software in the file.
    
    ```text
    Django>=2.0,<3.0
    psycopg2>=2.7,<3.0
    ```
    
10. Save and close the `requirements.txt` file.

11. Create a file called `docker-compose.yml` in your project directory.

    The `docker-compose.yml` file describes the services that make your app. In this example those services are a web server and database. The compose file also describes which Docker images these services use, how they link together, any volumes they might need mounted inside the containers. Finally, the `docker-compose.yml` file describes which ports these services expose. See the [`docker-compose.yml` reference][5] for more information on how this file works.

12. Add the following configuration to the file.
    
    ```dockerfile
    version: '3'
        
        services:
          db:
            image: postgres
          web:
            build: .
            command: python manage.py runserver 0.0.0.0:8000
            volumes:
              - .:/code
            ports:
              - "8000:8000"
            depends_on:
              - db
    ```

    This file defines two services: The `db` service and the `web` service.

13. Save and close the `docker-compose.yml` file.

### Create a Django project

In this step, you create a Django starter project by building the image from the build context defined in the previous procedure.

1. Change to the root of your project directory.

2. Create the Django project by running the [docker-compose run][6] command as follows.
    
    ```
    sudo docker-compose run web django-admin startproject composeexample .
    ```

    This instructs Compose to run `django-admin startproject composeexample` in a container, using the `web` service's image and configuration. Because the `web` image doesn't exist yet, Compose builds it from the current directory, as specified by the `build: .` line in `docker-compose.yml`.
    
    Once the `web` service image is built, Compose runs it and executes the `django-admin startproject` command in the container. This command instructs Django to create a set of files and directories representing a Django project.

3. After the `docker-compose` command completes, list the contents of your project.
    
    ```
    vagrant@ubuntu-bionic:/vagrant$ ls -l
    total 71
    -rwxrwxrwx 1 vagrant vagrant   145 Jan 17 22:20 Dockerfile
    -rwxrwxrwx 1 vagrant vagrant  1608 Jan 17 22:17 Vagrantfile
    drwxrwxrwx 1 vagrant vagrant     0 Jan 17 22:39 composeexample
    -rwxrwxrwx 1 vagrant vagrant   209 Jan 17 22:37 docker-compose.yml
    -rwxrwxrwx 1 vagrant vagrant   634 Jan 17 22:39 manage.py
    -rwxrwxrwx 1 vagrant vagrant    35 Jan 17 22:21 requirements.txt
    -rwxrwxrwx 1 vagrant vagrant 44375 Jan 17 22:18 ubuntu-bionic-18.04-cloudimg-console.log

    ```

### Connect the database

In this section, you set up the database connection for Django.

1. In your project directory, edit the `composeexample/settings.py` file.

2. Replace the `DATABASES = ...` with the following:
    
    ```python
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.postgresql',
            'NAME': 'postgres',
            'USER': 'postgres',
            'HOST': 'db',
            'PORT': 5432,
        }
    }
    ```
    
    These settings are determined by the [postgres][7] Docker image specified in `docker-compose.yml`.

3. Save and close the file.

4. Run the [sudo docker-compose up][8] command from the top level directory for your project.
    
    ```
    $ sudo docker-compose up
    djangosample_db_1 is up-to-date
    Creating djangosample_web_1 ...
    Creating djangosample_web_1 ... done
    Attaching to djangosample_db_1, djangosample_web_1
    db_1   | The files belonging to this database system will be owned by user "postgres".
    db_1   | This user must also own the server process.
    db_1   |
    db_1   | The database cluster will be initialized with locale "en_US.utf8".
    db_1   | The default database encoding has accordingly been set to "UTF8".
    db_1   | The default text search configuration will be set to "english".
    
    . . .
    
    web_1  | May 30, 2017 - 21:44:49
    web_1  | Django version 1.11.1, using settings 'composeexample.settings'
    web_1  | Starting development server at http://0.0.0.0:8000/
    web_1  | Quit the server with CONTROL-C.
    ```
    
    At this point, your Django app should be running at port `8000` on your Docker host. On Docker Desktop for Mac and Docker Desktop for Windows, go to `http://localhost:8000` on a web browser to see the Django welcome page. If you are using [Docker Machine][9], then `docker-machine ip MACHINE_VM` returns the Docker host IP address, to which you can append the port (`:8000`).

    ![Django example][10]

    > Note:
    > 
    > On certain platforms (Windows 10), you might need to edit `ALLOWED_HOSTS` inside `settings.py` and add your Docker host name or IP address to the list. For demo purposes, you can set the value to:
    > 
    > This value is **not** safe for production usage. Refer to the [Django documentation][11] for more information.

5. List running containers.

    In another terminal window, list the running Docker processes with the `sudo docker container ls` command.
    
    ```
    $ sudo docker ps
    CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
    a6cc66c5651e        vagrant_web         "python manage.py ru…"   25 minutes ago      Up 25 minutes       0.0.0.0:8000->8000/tcp   vagrant_web_1
    0e5af14abc14        postgres            "docker-entrypoint.s…"   44 minutes ago      Up 44 minutes       5432/tcp                 vagrant_db_1
    ```
    
6. Shut down services and clean up by using either of these methods:

    * Stop the application by typing `Ctrl-C` in the same shell in where you started it:
        
        ```
        Gracefully stopping... (press Ctrl+C again to force)
        Killing test_web_1 ... done
        Killing test_db_1 ... done
        ```
 
    * Or, for a more elegant shutdown, switch to a different shell, and run [sudo docker-compose down][12] from the top level of your Django sample project directory.

        ```
        vagrant@ubuntu-bionic:/vagrant$ sudo docker-compose down
        Stopping vagrant_web_1 ... done
        Stopping vagrant_db_1  ... done
        Removing vagrant_web_1                ... done
        Removing vagrant_web_run_6c6819d7113b ... done
        Removing vagrant_db_1                 ... done
        Removing network vagrant_default
        ```
        
Once you've shut down the app, you can safely remove the Django project directory (for example, `rm -rf django`).

## More Compose documentation

[documentation][13], [docs][14], [docker][15], [compose][16], [orchestration][17], [containers][18]

[1]: https://www.vagrantup.com/docs/vagrantfile/
[2]: https://docs.docker.com/get-started/
[3]: https://docs.docker.com/engine/reference/builder/
[4]: https://hub.docker.com/r/library/python/tags/3/
[5]: https://docs.docker.com/compose-file.md
[6]: https://docs.docker.com/compose/reference/run/
[7]: https://hub.docker.com/images/postgres
[8]: https://docs.docker.com/compose/reference/up/
[9]: https://docs.docker.com/machine/overview.md
[10]: https://docs.docker.com/compose/images/django-it-worked.png
[11]: https://docs.djangoproject.com/en/1.11/ref/settings/#allowed-hosts
[12]: https://docs.docker.com/compose/reference/down/
[13]: /search/?q=documentation
[14]: /search/?q=docs
[15]: /search/?q=docker
[16]: /search/?q=compose
[17]: /search/?q=orchestration
[18]: /search/?q=containers
