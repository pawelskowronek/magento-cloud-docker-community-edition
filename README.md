# Magento Cloud Docker - Community Edition

## Versions

Magento: `2.4.8-p3 Community Edition`  
Magento Cloud Docker: `1.4.5`

## Prerequisities:

- Docker
- Mutagen
- Magento's [access credentials](https://experienceleague.adobe.com/docs/commerce-operations/installation-guide/prerequisites/authentication-keys.html)

## (Apple Silicon Only) Build docker images locally

1. Clone magento cloud docker repository.
   ```bash
   git clone https://github.com/magento/magento-cloud-docker.git
   ```
2. Change current working directory to magento cloud docker.
   ```bash
   cd magento-cloud-docker
   ```
3. Build docker images.

   ```docker
   docker image build --platform=linux/arm64/v8 -t magento/magento-cloud-docker-php:8.4-cli-1.4.5 ./images/php/8.4-cli

   docker image build --platform=linux/arm64/v8 -t magento/magento-cloud-docker-opensearch:3-1.4.5 ./images/opensearch/3

   docker image build --platform=linux/arm64/v8 -t magento/magento-cloud-docker-nginx:1.28-1.4.5 ./images/nginx/1.28

   docker image build --platform=linux/arm64/v8 -t magento/magento-cloud-docker-varnish:7.7-1.4.5 ./images/varnish/7.7
   ```

## Install Magento Cloud Docker

### Initialise the project

1. Create directory where you want to install Magento Cloud Docker.
2. Change to the new project directory.
3. Create composer project based on `magento/project-community-edition`.
   Replace `<public-key>:<private-key>` with your [access credentials](https://experienceleague.adobe.com/docs/commerce-operations/installation-guide/prerequisites/authentication-keys.html)
   ```docker
   docker run --rm -v "$(pwd)":/m2 "magento/magento-cloud-docker-php:8.4-cli-1.4.5" composer create-project --repository-url=https://<public-key>:<private-key>@repo.magento.com/ magento/project-community-edition=2.4.8-p3 /m2
   ```
4. Add your [access credentials](https://experienceleague.adobe.com/docs/commerce-operations/installation-guide/prerequisites/authentication-keys.html) to the auth.json file.
   ```json
   {
   	"http-basic": {
   		"repo.magento.com": {
   			"username": "<public-key>",
   			"password": "<private-key>"
   		}
   	}
   }
   ```
5. Add the ece-tools and Cloud Docker for Commerce packages.
   ```docker
   docker run --rm -v "$(pwd)":/m2 "magento/magento-cloud-docker-php:8.4-cli-1.4.5" composer require --working-dir=/m2 --no-update --dev magento/ece-tools magento/magento-cloud-docker
   ```
6. Create the default configuration source file, .magento.docker.yml to build the Docker containers for the local environment.
   ```yml
   name: magento
   system:
     mode: 'developer'
   services:
     php:
       version: '8.4'
       extensions:
         enabled:
           - xsl
           - json
           - redis
     mysql:
       version: '11.4'
       image: 'mariadb'
     redis:
       version: '7.2'
       image: 'redis'
     opensearch:
       version: '3'
       image: 'magento/magento-cloud-docker-opensearch'
   hooks:
     build: |
       set -e
       php ./vendor/bin/ece-tools run scenario/build/generate.xml
       php ./vendor/bin/ece-tools run scenario/build/transfer.xml
     deploy: 'php ./vendor/bin/ece-tools run scenario/deploy.xml'
     post_deploy: 'php ./vendor/bin/ece-tools run scenario/post-deploy.xml'
   mounts:
     var:
       path: 'var'
     app-etc:
       path: 'app/etc'
     pub-media:
       path: 'pub/media'
     pub-static:
       path: 'pub/static'
   ```
7. Update project.
   ```docker
   docker run --rm -v "$(pwd)":/m2 "magento/magento-cloud-docker-php:8.4-cli-1.4.5" composer update --working-dir=/m2
   ```
8. In your local project root, generate the Docker Compose configuration file with mutagen used for synchronizing data in Docker.
   ```docker
   docker run --rm -v "$(pwd)":/m2 "magento/magento-cloud-docker-php:8.4-cli-1.4.5" /m2/vendor/bin/ece-docker build:compose --mode="developer" --sync-engine="mutagen"
   ```
9. Copy the default configuration DIST file to your custom configuration file and make any necessary changes.
   ```bash
   cp .docker/config.php.dist .docker/config.php
   ```
10. Build files to containers and run in the background.
    ```docker
    docker compose up -d
    ```
11. Start the file synchronisation.
    When using docker compose instead of docker-compose ensure that contents of mutagen.sh were updated accordingly as they are still using old format of docker compose.

    ```bash
    bash ./mutagen.sh
    ```

12. Make sure file synchronisation has completed.
    ```mutagen
    mutagen sync list
    ```
    Check for status of each sync, when finished it will say under each sync session:
    ```
    Status: Watching for changes
    ```

### Install Magento Cloud Docker

1. Deploy Adobe Commerce in the Docker container.
   ```docker
   docker compose run --rm deploy cloud-deploy
   ```
2. Run post-deploy hooks.
   ```docker
   docker compose run --rm deploy cloud-post-deploy
   ```
3. Configure and connect Varnish.
   ```docker
   docker compose run --rm deploy magento-command config:set system/full_page_cache/caching_application 2 --lock-env
   ```
   ```docker
   docker compose run --rm deploy magento-command setup:config:set --http-cache-hosts=varnish --no-interaction
   ```
4. Clear the cache.
   ```docker
   docker compose run --rm deploy magento-command cache:clean
   ```

Access the local storefront by opening the following URL in a browser:  
https://magento2.docker

Access the local admin dashboard by opening the following URL in a browser:
https://magento2.docker/admin

## (Optional) Install Sample Data

```docker
docker compose run --rm deploy magento-command sampledata:deploy
```

```docker
docker compose run --rm deploy magento-command setup:upgrade
```

## (Optional) Install Hyva Theme

Now that Hyva Theme is officially open-sourced you can easily install it.

### Create account and get credentials

Get a free key by registering an account at hyva.io and creating one from your account dashboard.

### Add credentials to your project

```docker
docker compose run --rm deploy composer config repositories.private-packagist composer https://hyva-themes.repo.packagist.com/magento2-docke-XXXXX/
docker compose run --rm deploy composer config --global --auth http-basic.hyva-themes.repo.packagist.com token XXXXX
```

### Install Hyva Theme with Luma Checkout

Unfortunately Hyva Checkout is still a paid-for add-on so if you want to have a functional checkout you will need to install Luma Checkout fallback.

```docker
docker compose run --rm deploy composer require hyva-themes/magento2-default-theme hyva-themes/magento2-luma-checkout
docker compose run --rm deploy magento-command mo:di Magento_TwoFactorAuth Magento_AdminAdobeImsTwoFactorAuth
docker compose run --rm deploy magento-command setup:upgrade
```

### Enable Hyva Theme in Magento Dashboard

Navigate to the Content > Design > Configuration admin section and activate the hyva/default theme on a Website level.

### Clear caches

```docker
docker compose run --rm deploy magento-command cache:clean
docker compose run --rm deploy magento-command cache:flush
```

---

Based on:
https://developer.adobe.com/commerce/cloud-tools/docker/deploy/developer-mode/
https://docs.hyva.io/hyva-themes/getting-started/index.html#getting-started
