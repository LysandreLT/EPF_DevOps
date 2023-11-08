# EPF_DevOps
## TP1

### Question 1.1 : 
> *Document your database container essentials: commands and Dockerfile.*

First, we pull the postgres and adminer images : 
```
docker pull postgres
docker pull adminer
```

Then, we launch adminer and build/run the image with a network :
```
docker run -p "8090:8080" --network --name=adminer -d adminer
docker build . -t mydb
docker network create app-network
docker run -d --network app-network --name db mydb
```

Finaly, we add the following two lines at the end of the Dockerfile in order to init the database with some data :
```
COPY CreateScheme.sql /docker-entrypoint-initdb.d
COPY InsertData.sql /docker-entrypoint-init-db.d
```

### Question 1.2 :
> *1-2 Why do we need a multistage build? And explain each step of this dockerfile.*

A multistage build in Docker is a technique used to optimize the construction of Docker images, especially when building applications or services that involve multiple stages of development, 
such as compiling code, running tests, and packaging artifacts. The primary reason for using a multistage build is to reduce the size of the final Docker image while ensuring that only the 
necessary artifacts and dependencies are included. This helps improve the security, performance, and manageability of the image.

The Dockerfile have 2 steps :
- We build the application by compiling with Maven and Amazon Correto. Then, we copy the pom.xml file and the src directory into the container.
- We run the application by copying the .jar file created in the previous step and running it.

### Question 1.3 :
> *Document docker-compose most important commands.*
We have created a new http folder with a Dockerfile composed of the following 2 lines :
```
FROM httpd:2.4
COPY ./index.html /usr/local/apache2/htdocs/
```
We then built the corresponding image and launched the container with the following commands :
```
docker build -t myapache2 .
docker run -d --name my-front-app --network app-network -p 8080:80 my-apache2
docker cp my-front-app:/usr/local/apache2/conf/httpd.conf httpd.conf
```
Finally, we added a 3rd line to the Dockerfile that add the web page :
```
COPY ./httpd.conf /usr/local/apache2/conf/httpd.conf
```
With a rebuild and rerun of the image, the docker-compose command will work.

### Question 1.4 :
> *Document your docker-compose file.*

Here is the docker-compose.yml file :

```
version: '3.7'

services:
  backend:
    container_name: my-running-app
    build:
      context: /backend
    networks:
      - my-network
    depends_on:
      - database

  database:
    container_name: db
    image: mydb:latest
    networks:
      - my-network

  httpd:
    build:
      context: http
    ports:
      - "8080:80"
    networks:
      - my-network
    depends_on:
      - backend

networks:
  my-network:
```

 This Docker Compose file sets up three services: backend, database, and httpd. The backend service depends on the database service, and the httpd service depends on the backend service. 
 All services are connected to the custom network my-network, enabling them to communicate with each other. The services are built using Dockerfiles located in specific contexts, 
 and ports are mapped for the httpd service to allow external access to port 8080 on the host. The container_name property is used to provide custom names for the containers created from these services.

### Question 1.5 :
> *Document your publication commands and published images in dockerhub.*

After having used the ```docker compose up``` command, every services are built and launched. Them, for each image, we add a tag with the command ```docker tag```. Finaly, we publish the images on Dockerhub
with the command ```docker push```. I applied these guidelines to 3 images, one for each service:
- frontend
- database
- backend

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
## TP2
### Question 2.1 :
> *What are testcontainers?*

They simply are Java Open Source libraries that allow you to run a bunch of docker containers while testing.

### Question 2.2 :
> *Document your Github Actions configurations.*

Here is the Github Actions configurations file (main.yml) :

```
name: CI devops 2023

on:
  push:
    branches:
      - main
      - develop
  pull_request:

jobs:
  test-backend:
    runs-on: ubuntu-22.04
    steps:
      - name: Checkout Code
        uses: actions/checkout@v2.5.0
        with:
          fetch-depth: 0

      - name: Set up JDK 17
        uses: actions/setup-java@v2
        with:
          java-version: 17
          distribution: 'adopt'

      - name: Build and Test with Maven
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=LysandreLT_EPF_DevOps -Dsonar.organization=lysandrelt -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=${{ secrets.SONAR_TOKEN }}  --file ./backend/pom.xml

  # define job to build and publish docker image
  build-and-push-docker-image:
    needs: test-backend
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-22.04

    # steps to perform in job
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0

      - name: Login to DockerHub
        run: docker login -u ${{ secrets.LOGIN_DOCKER }} -p ${{ secrets.TOKEN_DOCKER }}

      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./backend
          # Note: tags has to be all lower-case
          tags: ${{secrets.LOGIN_DOCKER}}/backend
          # build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          context: ./database
          tags: ${{secrets.LOGIN_DOCKER}}/database
          # build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          context: ./http
          tags: ${{secrets.LOGIN_DOCKER}}/frontend
          # build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}
```

This GitHub Actions configuration defines two jobs. The first job, test-backend, is responsible for building and testing a Java application using Maven and SonarCloud. The second job, build-and-push-docker-image,
builds and pushes Docker images for different project components to DockerHub, but only on the main branch.
The workflow also specifies when it should run, such as on pushes to the main and develop branches and on pull requests. The jobs are orchestrated to ensure that the test-backend job runs before the 
build-and-push-docker-image job, ensuring that only successful builds are pushed to DockerHub.

### Question 2.3 :
> *Document your quality gate configuration.*

Here is a screen of the quality gate results : 

![Alt text](.github/workflows/Quality%20gate%20configuration.PNG?raw=true "quality gate results")

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
## TP3

### Question 3.1 :
> *Document your inventory and base commands.*

In the inventory, there is the following *setup.yml* file :

```
all:
  vars:
    ansible_user: centos
    ansible_ssh_private_key_file: EPF_DevOps/ansible/id_rsa
  children:
    prod:
      hosts: lysandre.letohic.takima.cloud
```

I used the following base commands :

- ```ansible all -i inventories/setup.yml -m ping``` to test the inventory and the server connection.
- ```ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"``` to instruct Ansible to run the setup module on all hosts specified in the setup.yml inventory and collect information
about their operating system distribution.
- ```ansible all -i inventories/setup.yml -m yum -a "name=httpd state=absent" --become``` to instruct Ansible to ensure that the httpd package is not installed on all hosts specified in the setup.yml inventory
file by using the yum module. If the package is already absent, Ansible will take no action, but if it is present, Ansible will remove it from the target hosts. The --become flag is used to ensure that Ansible
has the necessary privileges to perform this task.

### Question 3.2 :
> *Document your playbook*

Here is the *playbook.yml* file :
```
- hosts: all
  gather_facts: false
  become: true

  # Install Docker
  tasks:

    - name: Install device-mapper-persistent-data
      yum:
        name: device-mapper-persistent-data
        state: latest

    - name: Install lvm2
      yum:
        name: lvm2
        state: latest

    - name: add repo docker
      command:
        cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

    - name: Install Docker
      yum:
        name: docker-ce
        state: present

    - name: Make sure Docker is running
      service: name=docker state=started
      tags: docker
```

This Ansible playbook is intended to be applied to all hosts (as indicated by hosts: all). It installs Docker and its dependencies, adds the Docker repository configuration, installs Docker CE, and ensures that
the Docker service is running. The tasks are executed with elevated privileges (become: true) to perform actions that require administrative permissions. Additionally, tasks are given human-readable names and,
in some cases, tags for better organization and manageability.

### Question 3.3 :
> *Document your docker_container tasks configuration.*

In this section, we've modified the ``playbook.yml`` file to add roles for each task (docker, network, database, app and proxy) :
```
- name: Deploy My Application
  hosts: all
  gather_facts: false
  become: yes
  roles:
    - install-docker
    - create-network
    - launch-database
    - launch-app
    - launch-proxy
```

We then created a *roles* folder with packages for each role, using the command ```ansible-galaxy init```. 
For the *install-docker* role, the contents of the file allow Docker to be installed automatically :

```
- name: Clean packages
  command:
    cmd: yum clean -y packages

- name: Install device-mapper-persistent-data
  yum:
    name: device-mapper-persistent-data
    state: latest

- name: Install lvm2
  yum:
    name: lvm2
    state: latest

- name: add repo docker
  command:
    cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

- name: Install Docker
  yum:
    name: docker-ce
    state: present

- name: Make sure Docker is running
  service: name=docker state=started
  tags: docker
```

For the *create-network* role, the following code creates the network used by Docker :
```
- name: Create Docker Network
  docker_network:
    name: app-network
    state: present
```

For the *launch-database* role, the following code launches the database, the corresponding Docker container and creates the corresponding image :
```
- name: Launch database
  docker_container:
    name: db
    image: chambellan/my-database:1.0
    networks:
      - name: app-network
```

For the *launch-app* role, the following code launches the backend, the container and the corresponding image :
```
- name: Launch app
  docker_container:
    name: my-running-app
    image: chambellan/backend:1.0
    networks:
      - name: app-network
```

For the *launch-proxy* role, the following code launches the frontend on port 80, the container and the corresponding image :
```
- name: Run HTTPD
  docker_container:
    name: http
    ports:
      - "80:80"
    image: chambellan/frontend:latest
    networks:
      - name: app-network
```
