---
title: Extend the Docker configuration
description: Learn how to extend your Cloud Docker for Commerce configuration using multiple compose files.
---

# Extend the Docker configuration

You can use the built-in Docker extension mechanism to specify [multiple compose files][].

The following example replaces the default value of the `ENABLE_SENDMAIL` environment variable.

1. Create a `docker-compose-dev.yml` file inside your project root directory and add the following content:

   ```yaml
   version: '2'
   services:
     deploy:
       environment:
         - ENABLE_SENDMAIL=false
   ```

1. Pass both configuration files while executing your commands. For example:

   ```bash
   docker compose -f docker-compose.yml -f docker-compose-dev.yml run deploy bash
   ```

## Specify Docker build sources

To test changes to images or make more extensive changes to the containers, you must build them from source. You can do this by adding the `build:context` configuration to the `docker-compose.override.yml` file.

The following example defines the build context for the Web container. You can use the same technique to build from any of the images in  the `vendor/magento/magento-cloud-docker` directory or any other Docker image, including local images that are resourced outside the project.

```yaml
version: '2.1'
services:
  web:
    build:
      context: ./vendor/magento/magento-cloud-docker/images/nginx/1.9/
```

Use the `--force-recreate` option to refresh the container build to update the container configuration and test iteratively:

```bash
docker compose up -d --force-recreate --build
```

## Add a version of existing service

In the `magento/magento-cloud-docker` package, the available [service versions](../containers/index.md#service-configuration-options) are determined by the Docker service images configured in the `images` directory. You add a service version by creating a directory for the version and adding a `Dockerfile` and other files to configure the new version.

**To add a service version using a** `Dockerfile`:

1. Clone the `magento/magento-cloud-docker` project to your local environment if necessary.

1. On the command line, change to the directory that contains the existing service version configurations.

   ```bash
   cd magento-cloud-docker/images/<service-name>
   ```

1. Create a directory for the new version.

1. Change to the new directory.

1. Create a `Dockerfile` with any additional configuration details for the new version, such as supported plugins. You can use the `Dockerfile` from the previous version as a template.

1. Add the `docker-entrypoint.sh` and `healthcheck` files if needed.

1. Add any necessary `.conf` and `.ini` files for the service to the `etc` directory.

1. Build the image.

   ```bash
   docker build -t test/<service-name>:<service-version>
   ```

1. After the build succeeds, test the changes by specifying the [Docker build sources](#specify-docker-build-sources).

## Add a PHP extension

You can add PHP extensions to the PHP container by adding the extension configuration to the [ExtensionResolver.php][] configuration file for Cloud Docker for Commerce.

**To add a PHP extension**:

1. Clone the `magento/magento-cloud-docker` project to your local environment.

1. On the command line, navigate to the PHP directory.

   ```bash
   cd magento-cloud-docker/src/Compose/Php
   ```

1. Add one or more extensions to the `ExtensionResolver.php` file:

   -  Open the `ExtensionResolver.php` file for editing.

   -  Specify the required extension in the `getConfig` method by specifying the extension type and dependency.

      For example, the following block adds the `bcmath` extension:

      ```php?start_inline=1
      public static function getConfig(): array
      ...

      'bcmath' => [
          '>=7.0' => [self::EXTENSION_TYPE => self::EXTENSION_TYPE_CORE],
       ],
       ...
      ```

      In this case, the `bcmath` PHP core extension installs from `docker-php-source` images.

   <!-- <InlineAlert variant="info" slots="text"/> -->

   The configuration you specify depends on the location of the extension source files and method of installation. You can add PHP core extensions from the official Docker PHP images, from the PECL repository, or using an installation script. For details on the configuration attributes and format for the `getConfig` method, see [PHP extension configuration reference](#php-extension-configuration-reference).

1. Enable the extension by default, or by adding it to the `.magento.app.yaml` file:

   -  To enable an extension by default, add it to the `DEFAULT_PHP_EXTENSIONS` array in the `ExtensionResolver.php` file.

      ```php?start_inline=1

      /**
       * Extensions which should be installed by default
       */
      public const DEFAULT_PHP_EXTENSIONS = [
          'bcmath',
          'bz2',
          'calendar',
          'exif',
          'gd',
          'gettext',
          'intl',
          'mysqli',
          'pcntl',
          'pdo_mysql',
          'soap',
          'sockets',
          'sysvmsg',
          'sysvsem',
          'sysvshm',
          'opcache',
          'zip',
      ];
      ```

   -  If you add the extension to the `.magento.app.yaml` for your Cloud project, you must regenerate the Docker Compose configuration file and restart the Docker container.

1. Add any required `.ini` files to the PHP FPM container configuration.

   -  On the command line, navigate to the FPM image directory `magento-cloud-docker/images/php/fpm`:

      ```bash
      cd ../../../images/php/fpm
      ```

   -  Add each required `.ini` file to the `etc` directory.

   -  For each `.ini` file that you add, you must add the following line to the `Dockerfile` (`magento-cloud-docker/images/php/fpm/Dockerfile`):

      ```conf
      COPY etc/<filename>.ini /usr/local/etc/php/conf.d/<filename>.ini
      ```

1. Add any required `.ini` files to the PHP CLI container configuration.

   -  On the command line, navigate to the CLI image directory `magento-cloud-docker/images/php/cli`.

      ```bash
      cd ../cli
      ```

   -  Add each required `.ini` file to the `etc` directory.

   -  For each `.ini` file that you add, you must add the following line to the `Dockerfile` (`magento-cloud-docker/images/php/cli/Dockerfile`):

      ```conf
      ADD etc/<file-name>.ini /usr/local/etc/php/conf.d/<filename>.ini
      ```

1. Generate an updated `Dockerfile` for all PHP image versions included in the Cloud Docker for Commerce package.

   ```bash
   bin/ece-docker image:generate:php
   ```

1. Test the extension by specifying the [Docker build sources](#specify-docker-build-sources).

### PHP extension configuration reference

Use the following attributes to specify the PHP extension configuration in the `getConfig` method in the [ExtensionResolver.php][] file. The configuration you specify depends on method of installation: from the official Docker PHP images, from the PECL repository, or using an installation script.

| Configuration option | Description |
| -------------------- | ------------ |
| PHP version constraint | Specifies the extension versions to install. If different versions have different installation requirements, you must add the configuration for each version. |
| `EXTENSION_TYPE_CORE` | Extension that can be installed from a `docker-php-source` image. |
| `EXTENSION_TYPE_PECL` | Extensions that can be installed from the [PECL][] repository. |
| `EXTENSION_TYPE_INSTALLATION_SCRIPT` | For extensions that install using a command sequence. |
| `EXTENSION_TYPE` | Specifies whether the extension installed from the Docker PHP images, the PECL repository, or using an installation script. Valid values: `EXTENSION_TYPE_CORE`, `EXTENSION_TYPE_PECL`, or `EXTENSION_TYPE_INSTALLATION_SCRIPT`<br/>`EXTENSION_OS_DEPENDENCIES` | For PHP core or PECL extensions, specifies Linux package dependencies. These packages install in the order listed before installing the extension. |
`EXTENSION_CONFIGURE_OPTIONS` | For PHP core extensions, specifies any configuration options to apply when Docker configures the PHP extension using the `docker-php-ext-configure` command. |
`EXTENSION_PACKAGE_NAME` | Specifies the extension package name. This value is used to generate the installation command. |
`EXTENSION_INSTALLATION_SCRIPT` | For extension type `EXTENSION_TYPE_INSTALLATION_SCRIPT`, specifies the Bash script to install the extension. |

<InlineAlert variant="info" slots="text"/>

For information about extension types and extension installation, see the **How to install more PHP extensions** section of the [PHP, Docker Official Images][] page in _Docker Hub_.

### Example: Core extension configuration

The following example shows the configuration for adding the PHP core extension `gd` in the `ExtensionResolver.php` file.

```php?start_inline=1
public static function getConfig(): array
...
            'gd' => [
                '>=7.0 <=7.3' => [
                    self::EXTENSION_TYPE => self::EXTENSION_TYPE_CORE,
                    self::EXTENSION_OS_DEPENDENCIES => ['libjpeg62-turbo-dev', 'libpng-dev', 'libfreetype6-dev'],
                    self::EXTENSION_CONFIGURE_OPTIONS => [
                        '--with-freetype-dir=/usr/include/',
                        '--with-jpeg-dir=/usr/include/'
                    ],
                ],
                '>=7.4' => [
                    self::EXTENSION_TYPE => self::EXTENSION_TYPE_CORE,
                    self::EXTENSION_OS_DEPENDENCIES => ['libjpeg62-turbo-dev', 'libpng-dev', 'libfreetype6-dev'],
                    self::EXTENSION_CONFIGURE_OPTIONS => [
                        '--with-freetype=/usr/include/',
                        '--with-jpeg=/usr/include/'
                    ],
                ],

            ],
...
```

#### Example: PECL extension configuration

The following example shows the configuration for adding the `gnupg` extension from the [PECL][] repository.

```php?start_inline=1
public static function getConfig(): array
...
            'gnupg' => [
                '>=7.0' => [
                    self::EXTENSION_TYPE => self::EXTENSION_TYPE_PECL,
                    self::EXTENSION_OS_DEPENDENCIES => ['libgpgme11-dev'],
                ],
            ],
...
```

### Example: Configuration for extension installed using a script

The following example shows the configuration for installing the `ioncube` extension using an installation script.

```php?start_inline=1
public static function getConfig(): array
...

            'ioncube' => [
                '>=7.0 <=7.3' => [
                    self::EXTENSION_TYPE => self::EXTENSION_TYPE_INSTALLATION_SCRIPT,
                    self::EXTENSION_INSTALLATION_SCRIPT => <<< BASH
cd /tmp
curl -O https://downloads.ioncube.com/loader_downloads/ioncube_loaders_lin_x86-64.tar.gz
tar zxvf ioncube_loaders_lin_x86-64.tar.gz
export PHP_VERSION=$(php -r "echo PHP_MAJOR_VERSION.'.'.PHP_MINOR_VERSION;")
export PHP_EXT_DIR=$(php-config --extension-dir)
cp "./ioncube/ioncube_loader_lin_\${PHP_VERSION}.so" "\${PHP_EXT_DIR}/ioncube.so"
rm -rf ./ioncube
rm ioncube_loaders_lin_x86-64.tar.gz
BASH
                ],
            ],
...
```

<!-- link definitions -->

[ExtensionResolver.php]: https://github.com/magento/magento-cloud-docker/tree/develop/src/Compose/Php
[PECL]: https://pecl.php.net/
[PHP, Docker Official Images]: https://hub.docker.com/_/php
[multiple compose files]: https://docs.docker.com/compose/reference/overview/#specifying-multiple-compose-files
