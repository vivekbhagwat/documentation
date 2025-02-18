---
description: >-
  Step by step how to install Aidbox.Dev on-premises. Estimated reading time: 7
  min
---

# Setup Aidbox.Dev

### Overview

You can run Aidbox.Dev on macOS, Linux and Windows 10 Pro

We are providing a lightweight version of [Aidbox](https://www.health-samurai.io/aidbox) named **Aidbox.Dev** — a special version aimed at local development and in your CI. To obtain access to Aidbox.Dev, please use our [License server](https://license-ui.aidbox.app/).

### License obtaining

1.  Visit and register on the [License server](https://license-ui.aidbox.app) then click the `GET LICENSE KEY` button.

![](../.gitbook/assets/scr-2018-10-31_15-08-05.png)

2. Enter a short description of your project and select the `Aidbox.Dev` option.

![](../.gitbook/assets/screen-shot-2018-10-02-at-17.28.09.png)

3. Congratulations! Now you have a license key.

![](../.gitbook/assets/screen-shot-2018-10-02-at-17.34.31.png)

### Aidbox.Dev Installation

#### Step 1. Install Docker Compose

The recommended way to work with **Aidbox.Dev** is Docker Compose.  
Install the Docker for your OS following the [Get Docker page](https://docs.docker.com/install/), then go to instructions on the [installing Compose](https://docs.docker.com/v17.09/compose/install/#install-compose) page. 

#### Step 2. Create docker-compose.yaml

Clone our[ official repository](https://github.com/Aidbox/devbox) with the sample configuration.

```bash
$ git clone https://github.com/Aidbox/devbox.git
```

Create the .env file with your Aidbox.Dev parameters.

```text
$ cd devbox && cp .env.tpl .env
```

{% hint style="info" %}
docker-compose.yaml file is parameterized with environment variables that can be stored by convention in the .env file \(read more about [env & docker-compose](https://docs.docker.com/compose/environment-variables/)\).
{% endhint %}

{% code-tabs %}
{% code-tabs-item title="docker-compose.yaml" %}
```yaml
version: '3.1'
services:
  devbox:
    image: "${AIDBOX_IMAGE}"
    depends_on:
      - "devbox-db"
    links:
      - "devbox-db:database"
    ports:
      - "${AIDBOX_PORT}:${AIDBOX_PORT}"
    env_file:
      - .env
    environment:
      PGHOST: database
  devbox-db:
    image: "${PGIMAGE}"
    ports:
      - "${PGHOSTPORT}:${PGPORT}"
    volumes:
    - "./pgdata:/data"
    environment:
      POSTGRES_USER:     "${PGUSER}"
      POSTGRES_PASSWORD: "${PGPASSWORD}"
      POSTGRES_DB:       "${PGDATABASE}"
```
{% endcode-tabs-item %}
{% endcode-tabs %}

#### Step 3. **Fill in the .env file with the following parameters**

Open the `.env` file and insert your `License ID` and `License KEY,`specify FHIR version and decide about box security and distribution channel \( latest vs edge\).

{% code-tabs %}
{% code-tabs-item title=".env" %}
```bash
AIDBOX_LICENSE_ID=<your-license-id>
AIDBOX_LICENSE_KEY=<your-license-key>

AIDBOX_CLIENT_ID=root
AIDBOX_CLIENT_SECRET=secret

AIDBOX_ADMIN_ID=admin
AIDBOX_ADMIN_PASSWORD=password

AIDBOX_PORT=8888
AIDBOX_FHIR_VERSION=3.0.1

PGPORT=5432
PGHOSTPORT=5437
PGUSER=postgres
PGPASSWORD=postgres
PGDATABASE=devbox

PGIMAGE=aidbox/db:11.1.0

AIDBOX_IMAGE=healthsamurai/devbox:0.4.1
# uncomment next line to work with edge channel
# AIDBOX_IMAGE=healthsamurai/devbox:edge
```
{% endcode-tabs-item %}
{% endcode-tabs %}

| Variable | required | Desc |
| :--- | :--- | :--- |
| AIDBOX\_LICENSE\_ID | true | Your license ID |
| AIDBOX\_LICENSE\_ID | true | Your license key |
| AIDBOX\_CLIENT\_ID | false | Root Client ID \* \(default: root\) |
| AIDBOX\_CLIENT\_SECRET | false | Root Client Secret \* |
| AIDBOX\_FHIR\_VERSION | true | Version of FHIR - 1.0.2, 1.4.0, 1.8.0, 3.01, 3.2.0, 3.3.0; Currently 3.0.1 is recommended version. |
| AIDBOX\_IMAGE | true | can be specific image tag or one of :edge or :latest |
| AIDBOX\_ADMIN\_ID | false | Id for admin user \(default: admin\) |
| AIDBOX\_ADMIN\_PASSWORD | false | Password for admin user |

{% hint style="info" %}
If **AIDBOX\_CLIENT\_ID** & **AIDBOX\_CLIENT\_SECRET** are provided - Aidbox will start in secured mode with access control turned on; API Client with provided ID and secret will be created as well Access Policy, which grants root privileges to this Client. In a simplest way you can access Aidbox API using basic auth and this client credentials.
{% endhint %}

{% hint style="warning" %}
Starting Aidbox in open mode without AIDBOX\_CLIENT\_SECRET is disabled! 
{% endhint %}

{% hint style="info" %}
**AIDBOX\_IMAGE** specify one of image from **healthsamurai/devbox** repository. Take a look at available versions at [docker hub](https://hub.docker.com/r/healthsamurai/devbox/tags).
{% endhint %}

### Run Aidbox.Dev

#### **Run Docker Compose**

```bash
$ docker-compose up -d
```

That's it! Aidbox.Dev is running and you can point your browser to [http://localhost:8888](http://localhost:8888/) to see a fancy welcome page.   
  
****Insert your ADMIN\_ID and ADMIN\_PASSWORD and you will see your development environment. This is the beginning of the way to create your best healthcare products.

![](../.gitbook/assets/login-aidbox.gif)

{% hint style="info" %}
We welcome your questions and comments about our products. Have an idea/question you'd like to share with our community? [Submit it here](https://community.aidbox.app/)
{% endhint %}

### What next?

Learn how to obtain access to the [REST API](../tutorials/how-to-use-rest-api.md) by the link below.

{% page-ref page="../tutorials/how-to-use-rest-api.md" %}

### Advanced tips

You can test your docker-compose.yaml with

```bash
docker-compose config # to inspect your configuration
```

You can access you PostgreSQL  on localhost using $PGUSER and $PGPASSWORD from env variables on $PGHOSTPORT .

```bash
# source variables
$ set -o allexport && source .env && set +o allexport
# run psql
$ psql -h localhost -p $PGHOSTPORT
psql (10.3, server 11.1)
Type "help" for help.

devbox=# \dt
```

Or you can do it in db container:

```bash
$ docker-compose exec devbox-db psql devbox
psql (11.1)
Type "help" for help.

devbox=#
```

Inspect Aidbox logs:

```bash
$ docker-compose logs -f devbox
```

Use curl to access API 

```bash
$ curl localhost:$AIDBOX_PORT/\$metadata?_format=yaml
{message: Access Denied}
# ups box is secured
$ curl -u $AIDBOX_CLIENT_ID:$AIDBOX_CLIENT_SECRET \
  localhost:$AIDBOX_PORT/\$metadata?_format=yaml | less
```

{% hint style="info" %}
Be careful with **$** sign in url paths \(aka **/$metadata**\) in shell  - you have to escape it \( **/\$metadata**\) otherwise shell will try to interpret it as variables ;\)
{% endhint %}

#### Stop Aidbox.Dev

```bash
$ docker-compose stop
```

#### Destroy Aidbox.Dev

```bash
$ docker-compose down
```

By default docker-compose file mount folder `./pgdata` as persistent volume for PostgreSQL, so it will survive restarts and destroy of containers. If you want to completely cleanup previous installation just `rm -rf ./pgdata`

#### Upgrade Aidbox.Dev

```
$ docker-compose down
$ docker-compose pull
$ docker-compose up -d
```

