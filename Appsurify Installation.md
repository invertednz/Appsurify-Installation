# Installation guide

## Minimum Requirements

3 Server setup:
- PostgreSQL - min CPU-2 GB RAM- 8GB SSD -100GB
- RabbitMQ - min CPU-1 GB RAM-512 HDD-40GB
- WEB+WORKERs - min CPU-4 GB RAM-8GB SSD- 100GB'

*CentOS 7*
## Postgresql 10

1. Step-by-step from guide [https://www.postgresql.org/download/linux/redhat/](https://www.postgresql.org/download/linux/redhat/) or [https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-centos-7](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-centos-7)

###### Notes
Main config 
```bash
nano /var/lib/pgsql/10/data/postgresql.conf

```
Access config
```bash
nano /var/lib/pgsql/10/data/pg_hba.conf

```


## RabbitMQ
1. Install erlang >= 20 from this guide [https://github.com/rabbitmq/erlang-rpm](https://github.com/rabbitmq/erlang-rpm)
2. Step-by-Step from guide [https://www.rabbitmq.com/install-rpm.html](https://www.rabbitmq.com/install-rpm.html)


## Web Application

#### Build Docker Image
```bash
docker build --no-cache -f Dockerfile -t testbrain:latest .

```

#### Save image to file
```bash
docker save -o testbrain.tar testbrain:latest

```

#### Load image from file
```bash
docker load -i testbrain.tar

```



#### Create Docker Container from builded image


A list of all the environment variables needed to run the project. 
When working through Docker, it is possible to change them when creating a container before launching. 
Variables marked with * must be overridden, as in most cases their standard values are empty or not valid.

Requirement | Category | Variable Name | Default Value | Description
--- | --- | --- | --- | ---
No | SYSTEM | PYTHONUNBUFFERED | 1 | System variable.
No | SYSTEM | GIT_PYTHON_REFRESH | 'quiet' | System variable
Yes | ENVIRONMENT | ENV_NAME | 'web' | web - run web interface. worker - run celery workers.
Yes | ENVIRONMENT | DJANGO_SETTINGS_MODULE | 'testbrain.settings.standalone' | Define specify configuration file with Django Faramework variables.
Yes | DATABASE | RDS_DB_NAME | 'testbrain' | Define database name. 
Yes | DATABASE | RDS_USERNAME | 'postgres' | Define database username.
Yes | DATABASE | RDS_PASSWORD | '' | Define database password.
Yes | DATABASE | RDS_HOSTNAME | 'localhost' | Define database host/ip.
Yes | DATABASE | RDS_PORT | 5432 | Define database port.
Yes | RABBITMQ/CELERY | BROKER_URL | 'amqp://guest:guest@localhost:5672//' | Define RabbitMQ connection uri. 'amqp://<username>:<password>@<host>:<port>/<vhost>'
No | EMAIL NOTIFICATION | EMAIL_BACKEND | 'django.core.mail.backends.smtp.EmailBackend' | Define email send backend, see Django docs.
No | EMAIL NOTIFICATION | EMAIL_HOST | 'smtp.gmail.com' | Define mail server host. 
No | EMAIL NOTIFICATION | EMAIL_PORT | 587 | Define mail server port.
No | EMAIL NOTIFICATION | EMAIL_USE_TLS | 'True' | Define mail server TLS usage.
Yes | EMAIL NOTIFICATION | EMAIL_HOST_USER | '' | Define mail username.
Yes | EMAIL NOTIFICATION | EMAIL_HOST_PASSWORD | '' | Define mail user password.
No | EMAIL NOTIFICATION | DEFAULT_FROM_EMAIL | 'Noreply <noreply@appsurify.com>' | Define FROM string in messages.
No, only for amazon eb | AMAZON | REGION | '' | Define region specialy for Amazon deployment.
No, only for amazon eb | AMAZON | FILE_SYSTEM_ID | '' | Define EFS ID specialy for Amazon deployment.
Yes | ENVIRONMENT | MOUNT_DIRECTORY | '/app/efs' | Define path for storing cloned repositories.
Yes | ENVIRONMENT | BASE_SITE_NAME | 'Appsurify' | Define basically brand name.
Yes | ENVIRONMENT | BASE_SITE_DOMAIN | '' | Define base system domain.
Yes | ENVIRONMENT | BASE_ORG_DOMAIN | '' | Define base domain for organizations.
Yes | STRIPE | STRIPE_LIVE_MODE | False | Define True for use LIVE mode stripe integraiton.
Yes | STRIPE | STRIPE_LIVE_SECRET_KEY | 'sk_live_************************' | 
Yes | STRIPE | STRIPE_LIVE_PUBLIC_KEY | 'pk_live_************************' |
Yes | STRIPE | STRIPE_TEST_SECRET_KEY | 'sk_test_************************' |
Yes | STRIPE | STRIPE_TEST_PUBLIC_KEY | 'pk_test_************************' |
Yes | ENVIRONMENT | LOCAL_SECRET_KEY | '************************************************' | Define secret (random) string for webhooks integration.
Yes | ENVIRONMENT | GITHUB_SECRET_KEY | '***********************************************' | Define secret (random) string for webhooks integration.


##### Detailed information on the use of environment variables when creating a container can be found [here](https://docs.docker.com/engine/reference/commandline/create/).

#### Update docker image

**Stop and remove current containers**
Stop web and worker containers
```bash
docker stop testbrain_{web,worker}

```
Remove web and worker containers
```bash
docker rm testbrain_{web,worker}

```
Remove web and worker image from docker daemon storage
```bash
docker rmi testbrain_{web,worker}

```
After these actions, you can repeat the loading of the image and the creation of containers based on the new image.

#### WEB Panel Container

**Create container:**

```bash
docker create --name testbrain_web \
-p 80:80 \
--restart=always \
-e ENV_NAME='web' \
-e RDS_DB_NAME=testbrain \
-e RDS_USERNAME=postgres \
-e RDS_PASSWORD='' \
-e RDS_HOSTNAME=localhost \
-e BROKER_URL='amqp://guest:guest@testbrain_mq:5672//' \
-e DJANGO_SETTINGS_MODULE=testbrain.settings.standalone \
-e BASE_SITE_NAME='Appsurify' \
-e BASE_SITE_DOMAIN='' \
-e BASE_ORG_DOMAIN='domain.intranet' \
-v $(pwd)/var/storage:/app/efs \
-t testbrain:latest

```

**Start container:**

```bash
docker start testbrain_web

```

**First RUN**
```bash
docker exec testbrain_web python manage.py migrate --noinput
docker exec testbrain_web python manage.py auto_createsuperuser
docker exec -it testbrain_web python manage.py createorganization

```
* command `docker exec testbrain_web python manage.py auto_createsuperuser` add **superuser** with default credentials. Login: admin Password: adminadmin

**If run with mount src directory**
Need copy frontent files to project path.
```bash
docker exec testbrain_web cp /app/frontend/index.html /app/www/templates/
docker exec testbrain_web cp -r /app/frontend/. /app/www/static

```

####  WORKER Contaner

**Create container:**

```bash
docker create --name testbrain_worker \
--restart=always \
-e ENV_NAME='worker' \
-e RDS_DB_NAME=testbrain \
-e RDS_USERNAME=postgres \
-e RDS_PASSWORD='' \
-e RDS_HOSTNAME=localhost \
-e BROKER_URL='amqp://guest:guest@testbrain_mq:5672//' \
-e DJANGO_SETTINGS_MODULE=testbrain.settings.standalone \
-e BASE_SITE_NAME='Appsurify' \
-e BASE_SITE_DOMAIN='' \
-e BASE_ORG_DOMAIN='domain.intranet' \
-v $(pwd)/var/storage:/app/efs \
-t testbrain:latest

```

**Start container:**

```bash
docker start testbrain_worker

```


#### All in One
**Example of starting all services in the Docker environment**

```bash
# Create postgres service with alocated 1GB memory and connect(mount) directory for postgreSQL data storage.
docker create --name='testbrain_postgres' \
--restart=always \
--memory 1g \
--cpus 2 \
-v $(pwd)/var/lib/postgresql/data:/var/lib/postgresql/data \
-t postgres:10.3 \
postgres

docker start testbrain_postgres
docker exec -it testbrain_postgres createdb -h localhost -U postgres testbrain



# Create rabbitMQ service with allocated 512MB memory and connect(mount) directory for pesistent data storage.
docker create --name='testbrain_rabbitmq' \
--restart=always \
--memory 512m \
--cpus 1 \
-v $(pwd)/var/lib/rabbitmq:/var/lib/rabbitmq \
-t rabbitmq:3.7.14-management

docker start testbrain_rabbitmq




# Load testbrain image to docker
docker load -i testbrain.tar

# Create testbrain WEB container

docker create --name testbrain_web \
-p 80:80 \
--restart=always \
--link testbrain_postgres:testbrain_postgres \
--link testbrain_rabbitmq:testbrain_rabbitmq \
-e ENV_NAME='web' \
-e RDS_DB_NAME=testbrain \
-e RDS_USERNAME=postgres \
-e RDS_PASSWORD='' \
-e RDS_HOSTNAME=testbrain_postgres \
-e BROKER_URL='amqp://guest:guest@testbrain_rabbitmq:5672//' \
-e DJANGO_SETTINGS_MODULE=testbrain.settings.standalone \
-e BASE_SITE_NAME='Appsurify' \
-e BASE_SITE_DOMAIN='domain.intranet' \
-e BASE_ORG_DOMAIN='domain.intranet' \
-v $(pwd)/var/storage:/app/efs \
-t testbrain:latest

docker start testbrain_web

# If database is empty
docker exec testbrain_web python manage.py migrate --noinput
docker exec testbrain_web python manage.py auto_createsuperuser
docker exec -it testbrain_web python manage.py createorganization


# Create testbrain WORKER container

docker create --name testbrain_worker \
--restart=always \
--link testbrain_postgres:testbrain_postgres \
--link testbrain_rabbitmq:testbrain_rabbitmq \
-e ENV_NAME='worker' \
-e RDS_DB_NAME=testbrain \
-e RDS_USERNAME=postgres \
-e RDS_PASSWORD='' \
-e RDS_HOSTNAME=testbrain_postgres \
-e BROKER_URL='amqp://guest:guest@testbrain_rabbitmq:5672//' \
-e DJANGO_SETTINGS_MODULE=testbrain.settings.standalone \
-e BASE_SITE_NAME='Appsurify' \
-e BASE_SITE_DOMAIN='domain.intranet' \
-e BASE_ORG_DOMAIN='domain.intranet' \
-v $(pwd)/var/storage:/app/efs \
-t testbrain:latest

docker start testbrain_worker

```
