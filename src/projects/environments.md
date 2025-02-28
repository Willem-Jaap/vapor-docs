# Environments

[[toc]]

## Introduction

As you may have noticed, projects do not contain much information or resources themselves. All of your deployments and invoked commands are stored within environments. Each project may have as many environments as needed.

Typically, you will have an environment for "production", and a "staging" environment for testing your application. However, don't be afraid to create more environments for testing new features without interrupting your main staging environment.

## Creating Environments

Environments may be created using the `env` Vapor CLI command:

```bash
vapor env my-environment
```

This command will add a new environment entry to your project's `vapor.yml` file that you may deploy when ready:

```yaml
id: 2
name: vapor-laravel-app
environments:
    production:
        build:
            - 'composer install --no-dev'
    my-environment:
        build:
            - 'composer install --no-dev'
```

In addition to our native runtimes, Vapor supports Docker image deployments. If you would like an environment to use a Docker image runtime instead of the default Vapor runtime, use the `--docker` option when creating your environment:

```bash
vapor env my-environment --docker
```

This command will create a `my-environment.Dockerfile` file in your application's root directory.

## Opening Environments

Environments may be opened in your default browser using the Vapor CLI's `open` command:

```bash
vapor open my-environment
```

## Default Environment

When executing a Vapor CLI command, Vapor CLI uses the `staging` environment by default:

```bash
vapor open // Opens the `staging` environment in your default browser...

vapor open production // Opens the `production` environment in your default browser...
```

However, within your application's `vapor.yml` file, you may define a `default-environment` option to change the default environment for your project:

```yaml
id: 2
name: vapor-laravel-app
default-environment: production
environments:
    production:
        build:
            - 'composer install --no-dev'
    my-environment:
        build:
            - 'composer install --no-dev'
```

## Environment Variables

Each environment contains a set of environment variables that provide crucial information to your application during execution, just like the variables present in your application's local `.env` file.

### Vapor's Default Environment Variables

Vapor automatically injects a variety of environment variables based on your environment's configured cache, database, etc. As an example, by adding a `cache` or `database` key to your `vapor.yml` file, Vapor will inject the necessary `CACHE_*` and `DB_*` environment variables.

Here is the full list of environment variables injected by Vapor on your environment:

| `.env` Value            | `env()` Value                                              |
|-------------------------|------------------------------------------------------------|
| `APP_ENV`               | Environment name                                           |
| `APP_DEBUG`             | False                                                      |
| `APP_LOG_LEVEL`         | Debug                                                      |
| `APP_URL`               | Vanity domain, or custom domain if exists                  |
| `ASSET_URL`             | CloudFront                                                 |
| `AWS_BUCKET`            | Storage resource if exists                                 |
| `BROADCAST_DRIVER`      | Pusher                                                     |
| `CACHE_DRIVER`          | DynamoDB, or cache (Redis) resource if exists              |
| `DB_*`                  | Database (MySQL, Postgresql, etc) resource if exists       |
| `DYNAMODB_CACHE_TABLE`  | `vapor_cache`                                              |
| `FILESYSTEM_DISK`       | S3                                                         |
| `FILESYSTEM_DRIVER`     | S3                                                         |
| `FILESYSTEM_CLOUD`      | S3                                                         |
| `LOG_CHANNEL`           | Stderr                                                     |
| `MAIL_DRIVER`           | LOG, but SES for environments with the name `production`   |
| `MAIL_MAILER`           | LOG, but SES for environments with the name `production`   |
| `MAIL_FROM_ADDRESS`     | `hello@example.com` or `hello@custom-domain.com` if exists |
| `MAIL_FROM_NAME`        | `your_project_name`                                        |
| `MIX_URL`               | CloudFront                                                 |
| `QUEUE_CONNECTION`      | SQS                                                        |
| `SCHEDULE_CACHE_DRIVER` | DynamoDB                                                   |
| `SESSION_DRIVER`        | Cookie                                                     |

You will not see these environment variables when you manage your environment via Vapor CLI or Vapor UI, and any variables you manually define will override Vapor's automatically injected variables.

### Updating Environment Variables

You may update an environment's variables via the Vapor UI or using the `env:pull` and `env:push` CLI commands. The `env:pull` command may be used to pull down an environment file for a given environment:

```bash
vapor env:pull production
```

Once this command has been executed, a `.env.{environment}` file will be placed in your application's root directory. To update the environment's variables, simply open and edit this file. When you are done editing the variables, use the `env:push` command to push the variables back to Vapor:

```bash
vapor env:push production
```

If you are using the DotEnv library's [variable nesting](https://github.com/vlucas/phpdotenv#nesting-variables) feature to reference default environment variables that Vapor is injecting, you should replace these references with literal values instead. Since Vapor's injected environment variables do not belong to the environment file, they can not be referenced using the nesting feature.

:::tip Variables & Deployments

After updating an environment's variables, the new variables will not be utilized until the application is deployed again. In addition, when rolling back to a previous deployment, Vapor will use the variables as they existed at the time the deployment you're rolling back to was originally deployed.
:::

:::warning Environment Variable Limits

Due to AWS Lambda limitations, your environment variables may only be 4kb in total. You should use encrypted environment files in place of or in addition to environment variables if you exceed this limit.
:::

### Reserved Environment Variables

The following environment variables are reserved and may not be added to your environment:

- _HANDLER
- AWS_ACCESS_KEY_ID
- AWS_DEFAULT_REGION
- AWS_EXECUTION_ENV
- AWS_LAMBDA_FUNCTION_MEMORY_SIZE
- AWS_LAMBDA_FUNCTION_NAME
- AWS_LAMBDA_FUNCTION_VERSION
- AWS_LAMBDA_LOG_GROUP_NAME
- AWS_LAMBDA_LOG_STREAM_NAME
- AWS_LAMBDA_RUNTIME_API
- AWS_REGION
- AWS_SECRET_ACCESS_KEY
- AWS_SESSION_TOKEN
- LAMBDA_RUNTIME_DIR
- LAMBDA_TASK_ROOT
- TZ

In addition, environment variables should not contain `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, or `AWS_SESSION_TOKEN` in their names. For example: `MY_SERVICE_AWS_SECRET_ACCESS_KEY`.

## Encrypted Environment Files

Vapor provides built-in support for Laravel's [encrypted environment files](https://laravel.com/docs/9.x/configuration#encrypting-environment-files). If Vapor discovers an encrypted environment file while booting your application, it will automatically attempt to decrypt it and inject the resulting variables into the runtime.

To leverage this feature, you must first ensure an encrypted environment file is present at the root of your application during deployment. For example, deploying the `production` environment requires a file called `.env.production.encrypted` to be present at the root of your application.

Additionally, you should ensure the decryption key is available in the runtime by defining it as the `LARAVEL_ENV_ENCRYPTION_KEY` environment variable via the Vapor UI or [CLI](#environment-variables).

:::warning Version Requirements

Utilizing encrypted environment files requires your application to be running Laravel >= v9.37 and Vapor Core >= v2.26.
:::

### Passport Keys

You may easily add your project's Passport keys to your encrypted environment file using the `env:passport` CLI command:

```bash
vapor env:passport production
```

The `env:passport` command will append the contents of your local Passport keys to your `.env.production` file. When the command completes, you should re-encrypt the environment file using Laravel's `env:encrypt` command and redeploy your project in order for the changes to take effect.

## Maintenance Mode

When deploying a Laravel application using a traditional VPS like those managed by [Laravel Forge](https://forge.laravel.com), you may have used the `php artisan down` command to place your application in "maintenance mode". To place a Vapor environment in maintenance mode, you may use the Vapor UI or the `down` CLI command:

```bash
vapor down production
```

To remove an environment from maintenance mode, you may use the `up` command:

```bash
vapor up production
```

:::tip Maintenance Mode & Vanity URLs

When an environment is in maintenance mode, the environment's custom domain will display a maintenance mode splash screen; however, you may still access the environment via its "vanity URL"
:::

### Customizing The Maintenance Mode Screen

You may customize the maintenance mode splash screen for your application by placing a `503.html` file in your application's root directory. In addition, you may also place a `503.json` file in your application's root directory for requests asking for JSON responses.

### Bypassing Maintenance Mode

You may find it useful to be able to access your site on its custom domain rather than the vanity domain while in maintenance mode. To accomplish this, you may provide a secret when invoking the `down` command:

```bash
vapor down --secret="example-secret"
```

You may then access your application using the secret key as the URL path:

```
https://example.com/example-secret
```

## Commands

Commands allow you to execute an arbitrary Artisan command against an environment. You may issue a command via the Vapor UI or using the `command` CLI command. The `command` command will prompt you for the Artisan command you would like to run:

```bash
vapor command production

vapor command production --command="php artisan inspire"
```

By default, your command will timeout after one minute. You can configure the timeout of your CLI commands using the `cli-timeout` option within your `vapor.yml` file. This option allows you to specify the maximum number of seconds a CLI command should be allowed to run:

```yaml
id: 2
name: vapor-laravel-app
environments:
    production:
        cli-timeout: 20
        build:
            - 'composer install --no-dev'
```

Sometimes you may need to run the previous command again. To accomplish this, you may use the Vapor UI or the `command:again` CLI command:

```bash
# Run the previous command again...
vapor command:again

# Run a specific command again using the command's ID...
vapor command:again 50
```

## Memory

Vapor (via AWS Lambda) allocates CPU power to your Lambda function in proportion to the amount of memory configured for the application. You may increase or decrease the configured memory using the `memory` option in your environment's `vapor.yml` configuration:

```yaml
id: 2
name: vapor-laravel-app
environments:
    production:
        memory: 1024
        build:
            - 'composer install --no-dev'
```

:::warning Memory Increments

When configuring the memory for your Lambda function, you may define a value between 128 MB and 10,240 MB in 64-MB increments.
:::

## Concurrency

By default, Vapor will allow your application to process web requests at max concurrency, which is typically 1,000 requests executing at the same time at any given moment across all of your AWS Lambda functions in a given region. If you would like to reduce the maximum web concurrency, you may define the `concurrency` option in the environment's `vapor.yml` configuration. Additionally, if you need more than 1,000 concurrent requests, you can submit a limit increase request in the AWS Support Center console.

While maximum performance is certainly appealing, in some situations it may make sense to set this value to the maximum concurrency you reasonably expect for your particular application. Otherwise, a DDoS attack against your application could result in larger than expected AWS costs:

```yaml
id: 2
name: vapor-laravel-app
environments:
    production:
        concurrency: 50
        build:
            - 'composer install --no-dev'
```

## Prewarming

By default, when an environment is deployed, the first request it receives may encounter "cold starts". These requests typically incur a penalty of a few seconds while AWS loads a serverless container to serve the request. Once a request has been served by that container, it is typically kept warm to serve further requests with no delay.

To mitigate "cold starts" after a fresh deployment, Vapor allows you to define a `warm` configuration value for an environment in your `vapor.yml` file. The `warm` value represents how many serverless containers Vapor will "pre-warm" by making concurrent requests to the newly deployed application **before it is activated for public accessibility**. Vapor will continue to pre-warm this many containers every 5 minutes while the application is deployed so the specified number of containers are always ready to serve requests:

```yaml
id: 2
name: vapor-laravel-app
environments:
    production:
        warm: 10
        build:
            - 'composer install --no-dev'
```

## Firewall

You may instruct Vapor to automatically configure a firewall that provides basic protection against denial-of-service attacks targeting your environment, as well as protection against pervasive bot traffic that can consume your environment's resources.

Before getting started, keep in mind that Vapor's managed firewall inspects requests using the IP address from the web request origin. Therefore, this feature should only be used if the requests are not already being reversed proxied through a service such as Cloudflare. **If you are already using a reverse proxy, you should not use this feature**.

You may use Vapor's managed firewall by defining the `firewall` configuration option within your application's `vapor.yml` file:

```yaml
id: 2
name: vapor-laravel-app
environments:
    production:
        build:
            - 'composer install --no-dev'
        firewall:
            rate-limit: 1000
            bot-control:
                - CategorySearchEngine
                - CategorySocialMedia
                - CategoryScrapingFramework
```

### `rate-limit`

When using the `rate-limit` option, Vapor's managed firewall tracks the rate of requests for each originating IP address and blocks IPs with request rates over the given `rate-limit` value. In the example above, if the request count for an IP address exceeds 1,000 requests in any 5-minute time span then the firewall will temporarily block requests from that IP address with the `403 Forbidden` HTTP status code.

### `bot-control`

When using the `bot-control` option, Vapor's managed firewall blocks requests from pervasive bots, such as scrapers or search engines. You may customize the "category" of requests the `bot-control` should block by providing an `array` of categories within your application's `vapor.yml` file:

```yaml
firewall:
    bot-control:
        - CategoryAdvertising
        - CategoryArchiver
        - SignalNonBrowserUserAgent
```

Here is the list of available categories you may use:

| Category | Description |
| --- | --- |
| CategoryAdvertising | Blocks requests from bots that are used for advertising purposes\. |
| CategoryArchiver | Blocks requests from bots that are used for archiving purposes\. |
| CategoryContentFetcher | Blocks requests from bots that are fetching content on behalf of an end-user\. |
| CategoryHttpLibrary | Blocks requests from HTTP libraries that are often used by bots\. |
| CategoryLinkChecker | Blocks requests from bots that check for broken links\. |
| CategoryMiscellaneous | Blocks requests from miscellaneous bots\. |
| CategoryMonitoring | Blocks requests from bots that are used for monitoring purposes\. |
| CategoryScrapingFramework | Blocks requests from web scraping frameworks\. |
| CategorySecurity | Blocks requests from security\-related bots\. |
| CategorySeo | Blocks requests from bots that are used for search engine optimization\. |
| CategorySocialMedia | Blocks requests from bots that are used by social media platforms to provide content summaries\. Verified social media bots are not blocked\.  |
| CategorySearchEngine | Blocks requests from search engine bots\. Verified search engines are not blocked\.  |
| SignalAutomatedBrowser | Blocks requests with indications of an automated web browser\. |
| SignalKnownBotDataCenter | Blocks requests from data centers that are typically used by bots\. |
| SignalNonBrowserUserAgent | Blocks requests with user-agent strings that don't seem to be from a web browser\. |

---

:::warning API Gateway v2

Due to AWS limitations, Vapor's managed firewall does not support API Gateway v2.
:::

Behind the scenes, Vapor's managed firewall uses **[Amazon WAF](https://aws.amazon.com/waf/)**, creating a Web ACL with one rate-based rule per Vapor environment. Feel free to check out the AWS WAF documentation for more information about WAF and its pricing.

## Timeout

By default, Vapor will limit web request execution time to 10 seconds. If you would like to change the timeout value, you may add a `timeout` value (in seconds) to the environment's configuration. API Gateway has a maximum timeout of 30 seconds. Should you require a longer request duration, you may utilize a [load balancer](/resources/networks#load-balancers). Note that AWS does not allow Lambda executions to process for more than 15 minutes:

```yaml
id: 2
name: vapor-laravel-app
environments:
    production:
        timeout: 20
        build:
            - 'composer install --no-dev'
```

:::tip Detecting Timeouts In Logs

Remember, you can use the [Vapor UI dashboard package](/introduction#installing-the-vapor-ui-dashboard) to search for timeout occurrences.
:::

## Scheduler

Vapor automatically configures Laravel's task scheduler and instructs it to use the DynamoDB cache driver to avoid overlapping tasks, so no other configuration is required to begin leveraging Laravel's scheduled task feature.

:::warning Running Background Tasks

Due to the serverless nature of Vapor, you should avoid using the `runInBackground` method when scheduling jobs. Doing so may prevent other tasks from running in the event the Lambda container shuts down before the current task completes.
:::

If you would like to disable the scheduler, you may set an environment's `scheduler` option to `false`:

```yaml
id: 2
name: vapor-laravel-app
environments:
    production:
        scheduler: false
        build:
            - 'composer install --no-dev'
```

:::warning Log Messages

Due to Vapor limitations, log messages from scheduled tasks will not appear in AWS CloudWatch or Vapor UI. As a workaround, you should dispatch a queued job from your scheduled tasks and write log messages from your queued job.
:::

### Sub-Minute Scheduled Tasks

Although Laravel's [sub-minute scheduled tasks](https://laravel.com/docs/scheduling#sub-minute-scheduled-tasks) can run on Vapor, there are a few caveats to consider.

Due to AWS limitations, there is no guarantee the scheduler will be invoked at the very beginning of any given minute. Therefore, you may find that sub-minute tasks scheduled early in the `schedule:run` process do not run as expected and those which run later in the schedule may not start at the expected time.

For example, when scheduling a command using `everyThirtySeconds` and assuming the scheduler is invoked by AWS at 12:00:10, you should expect your command to run at 12:00:10 and 12:00:40.

In addition, the `runInBackground` option is not supported on Vapor; therefore, you may find some of your tasks are blocked from running if the previous task runs longer than expected.

## Mail

Laravel provides a clean, simple email API. And, by default, Vapor will automatically configure your environment to use **[Amazon SES](https://aws.amazon.com/ses/)** as the default mail driver by injecting the proper Laravel environment variables during deployment. Of course, you may change the default mail driver by defining a different value for the `MAIL_MAILER` environment variable.

If you plan to use Amazon SES as your application's mail service, you should first ensure the `MAIL_MAILER` environment variable is set and it contains the value `ses`. Next, you should [attach a domain](#custom-domains) to your environment. Once attached and deployed, Vapor will automatically update the domain's DNS records so Amazon SES can validate the domain and configure DKIM. These DNS records are necessary to protect your reputation as a sender.

:::warning Self-Managed Domains

If you self-manage your domain's DNS records, Vapor will not be able to update the domain's DNS records automatically. Therefore, you should run the `vapor record:list domain-name.com` command to view the records that Vapor indicates are required for your domain and update your DNS records accordingly.
:::

By default, Vapor configures the "From Address" and "From Name" Laravel configuration settings with `hello@your-domain.com` and `Your Project Name`, respectively. Of course, you're free to modify these values by defining new values for the `MAIL_FROM_ADDRESS` and `MAIL_FROM_NAME` environment variables.

Finally, if you haven't used Amazon SES before, your SES account will be in "sandbox" mode. Sandbox mode only allows you to send emails to manually verified domains. To move out of SES sandbox mode, follow these instructions: **[https://docs.aws.amazon.com/ses/latest/DeveloperGuide/request-production-access.html](https://docs.aws.amazon.com/ses/latest/DeveloperGuide/request-production-access.html)**.

## Metrics

A variety of environment performance metrics may be found in the Vapor UI or using the `metrics` CLI command:

```bash
vapor metrics production
vapor metrics production 5m
vapor metrics production 30m
vapor metrics production 1h
vapor metrics production 8h
vapor metrics production 1d
vapor metrics production 3d
vapor metrics production 7d
vapor metrics production 1M
```

### Alarms

You may configure alarms for all environment metrics using the Vapor UI. These alarms will notify you via the notification method of your choice when an alarm's configured threshold is broken and when an alarm recovers.

## Runtime

The `runtime` configuration option allows you to specify the runtime a given environment runs on.

### Native Runtimes

The currently supported native runtimes are `php-8.0:al2`, `php-8.1:al2`, `php-8.2:al2`, `php-8.2:al2-arm`, `php-8.3:al2`, and `php-8.3:al2-arm`. The runtimes that are suffixed with `al2` use Amazon Linux 2 while those without the suffix use Amazon Linux 1. Runtimes without the `-arm` suffix run on x86 artchitecture whereas those suffixed with `-arm` run on Arm architecture. Arm provides performance benefits and cost savings compared to its x86 equivalent:

```yaml
id: 2
name: vapor-laravel-app
environments:
    production:
        runtime: 'php-8.2:al2'
        build:
            - 'composer install --no-dev'
```

:::warning Amazon Linux 2

Using [Amazon Linux 2](https://aws.amazon.com/amazon-linux-2/) (`php-8.0:al2`, `php-8.1:al2`, `php-8.2:al2`, `php-8.2:al2-arm`, `php-8.3:al2`, `php-8.3:al2-arm`) is **highly recommended**, as Amazon Linux 1 is no longer maintained as of December 31, 2020.
:::

The following limitations apply to Vapor native runtimes:

- The application size, including the runtime itself, must not exceed 250MB.
- Additional PHP extensions or libraries (such as `imagick`) can not be installed.

#### Customizing Core `php.ini` Directives

To customize core `php.ini` directives on our native runtimes, create a `php.ini` file in the `php/conf.d` directory of your project with the desired directives. For example, to change the `upload_max_filesize`, add the following line to your application's `php.ini` file:

```ini
upload_max_filesize = 4M
```

After deploying, the new directive will take effect in your Vapor environment.

### Docker Runtimes

Docker based runtimes allow you to package and deploy applications up to 10GB in size and allow you to install additional PHP extensions or libraries by updating the environment's corresponding `.Dockerfile`. For every new Docker based environment, Vapor creates a `.Dockerfile` file within your application that uses one of Vapor's base images as a starting point for building your image. All of Vapor's Docker images are based on Alpine Linux:

```docker
FROM laravelphp/vapor:php82

COPY . /var/task
```

Vapor's Docker runtimes also support Arm architecture, which provides increased performance and cost savings. If you want to take advantage of Arm architecture, you should update your base image as follows:

```docker
FROM laravelphp/vapor:php82-arm

COPY . /var/task
```

If you would like to use a Docker image instead of the Vapor native runtimes, you should set your `vapor.yml` configuration file's `runtime` option to `docker` (for x86 architecture) or `docker-arm` (for Arm architecture):

```yaml
# x86
id: 2
name: vapor-laravel-app
environments:
    production:
        runtime: docker
        build:
            - 'composer install --no-dev'

# Arm
id: 2
name: vapor-laravel-app
environments:
    production:
        runtime: docker-arm
        build:
            - 'composer install --no-dev'
```

:::warning Building Docker Arm Images

To avoid errors when compiling an image for the `docker-arm` runtime, it is essential to ensure that the environment where the image is built is compatible with `arm64` based architecture. To avoid such issues, it may be necessary to emulate the `arm64` architecture using a tool such as QEMU. The [Docker documentation](https://docs.docker.com/build/building/multi-platform/#building-multi-platform-images) provides further guidance on this topic.
:::

If you would like to rename the Dockerfile or use a shared Dockerfile across multiple environments, you may define a `dockerfile` option within your `vapor.yml` file:

```yaml
id: 2
name: vapor-laravel-app
environments:
    production:
        runtime: docker
        dockerfile: vapor.Dockerfile
        build:
            - 'composer install --no-dev'
```

:::warning Migrating Existing Environments To A Docker Runtime

When migrating an existing environment to a Docker runtime, please keep in mind that you won't be able to revert that environment to the default Vapor Lambda runtime later. For that reason, you may want to create an environment for testing the Docker runtime first.
:::

Vapor will build, tag, and publish your environment's image during your deployments; therefore, you should ensure that you have installed [Docker](https://docs.docker.com/get-docker/) on your local machine. Vapor's base Docker images are:

- `laravelphp/vapor:php74`
- `laravelphp/vapor:php80`
- `laravelphp/vapor:php81`
- `laravelphp/vapor:php82`
- `laravelphp/vapor:php82-arm`
- `laravelphp/vapor:php83`
- `laravelphp/vapor:php83-arm`

Of course, you are free to modify your environment's `Dockerfile` to install additional dependencies or PHP extensions. Here are a few examples:

```docker
FROM laravelphp/vapor:php81

# Add the `ffmpeg` library...
RUN apk --update add ffmpeg

# Add the `mysql` client...
RUN apk --update add mysql-client

# Add the `gmp` PHP extension...
RUN apk --update add gmp gmp-dev
RUN docker-php-ext-install gmp

# Place application in Lambda application directory...
COPY . /var/task
```

#### Customizing Core `php.ini` Directives

To customize core `php.ini` directives on our native runtimes, create a `php.ini` file in the **root** directory of your project with the desired directives. For example, to change the `upload_max_filesize` directive, add the following line to your `php.ini` file:

```ini
upload_max_filesize = 4M
```

Next, in your Dockerfile, include a new `COPY` command that copies the local `php.ini` file into the Docker image:

```docker
FROM laravelphp/vapor:php81

COPY ./php.ini /usr/local/etc/php/conf.d/overrides.ini

COPY . /var/task
```

After deploying, the new directive will take effect in your Vapor environment.

#### Docker Build Arguments

When using Docker, is common to use `ARG` instructions in `.Dockerfile` files to define build-time variables:

```docker
ARG VERSION=php81

FROM laravelphp/vapor:${VERSION}

COPY . /var/task
``` 

You may set Docker build arguments via the `docker-build-args` configuration option within your application's `vapor.yml` file:

```yaml
id: 2
name: vapor-laravel-app
environments:
    production:
        runtime: docker
        docker-build-args:
            VERSION: php81
        build:
            - 'composer install --no-dev'
```

Alternatively, you may provide one or multiple `--build-arg` options to the `deploy` Vapor CLI command to specify the values of the build arguments:

```bash
vapor deploy --build-arg VERSION=php74 --build-arg KEY=value
```

## Octane

By default, when Lambda receives HTTP requests, Vapor sends those requests synchronously to PHP-FPM using the FastCGI Protocol. This means every incoming request spawns a PHP-FPM worker and boots your application. This approach ensures Vapor behaves exactly like a traditional web server.

If you wish to reduce the overhead involved in using PHP-FPM, you may opt-in to **Laravel [Octane](https://laravel.com/docs/8.x/octane)** support on Vapor. Octane can increase your application's performance by booting your application once, keeping it in memory, and then feeding that same application instance requests as they are received.

To get started, [install Laravel Octane](https://laravel.com/docs/8.x/octane#installation) in your project. After installing Octane, don't forget to review important [Octane documentation](https://laravel.com/docs/8.x/octane) topics such as [dependency injection](https://laravel.com/docs/8.x/octane#dependency-injection-and-octane) and [managing memory leaks](https://laravel.com/docs/8.x/octane#managing-memory-leaks).

Finally, you may instruct Vapor to use Octane by setting the `octane` configuration option within your application's `vapor.yml` file:

```yaml
id: 1
name: my-application
environments:
    staging:
        memory: 1024
        runtime: 'php-8.2:al2'
        octane: true
```

In addition, if your project uses a database, you may use the `octane-database-session-persist` and `octane-database-session-ttl` options to instruct Octane that database connections should be reused between requests:

```yaml
        database: my-database
        octane: true
        octane-database-session-persist: true
        octane-database-session-ttl: 10
```

- The `octane-database-session-persist` option indicates that database connections should persist between requests. The main purpose of this option is to reduce the overhead involved on creating a database connection on each request.
- The `octane-database-session-ttl` option allows specifying the time (in seconds) the Lambda container should stay connected to the database when the Lambda container is not being used. This option is only available for MySQL fixed size databases.

**We recommended that you specify an `octane-database-session-ttl` value**; otherwise, the Lambda container will stay connected to your database until the Lambda container gets destroyed. This may take several minutes and may result in your database becoming overwhelmed with active connections.

## Gateway Versions

By default, Vapor routes HTTP traffic to your serverless applications using AWS API Gateway v1 (REST APIs). Your application may run on either API Gateway v1 (REST APIs) or API Gateway v2 (HTTP APIs). By default, applications deploy using API Gateway v1 as it provides a fuller feature set such as Vapor's managed Firewall, and more.

However, API Gateway v2 offers a cost reduction per million requests to your application ($1.00 per million vs. $3.50 per million). If you would like to use API Gateway v2, you may specify the `gateway-version` configuration option for a given environment in your `vapor.yml` file:

```yaml
id: 2
name: vapor-laravel-app
environments:
    production:
        gateway-version: 2
        build:
            - 'composer install --no-dev'
```

### HTTP to HTTPS Redirection With API Gateway v2

If you choose to use API Gateway v2 and would like to support HTTP to HTTPS redirection, we currently suggest using [Cloudflare](https://cloudflare.com) as an external DNS provider for your Vapor application. Cloudflare not only provides DNS, but serves as a reverse proxy to your application and features an option for automatic HTTP to HTTPS redirection.

After a Vapor deployment is completed, Vapor will provide you with CNAME records for the domain(s) associated with your environment. These records will point the domain to your Lambda application. You should manually add these values as CNAME records within the Cloudflare DNS dashboard.

In addition, when using Cloudflare, you should set your Cloudflare SSL / TLS mode to "Full". The "Always Use HTTPS" configuration option may be found under the SSL / TLS menu's "Edge Certificates" tab.

## Custom VPCs

If you want to place your Lambda functions within a VPC that is not managed by Vapor, you may specify the `subnets` and `security-groups` configuration options for a given environment in your `vapor.yml` file:

```yaml
id: 2
name: vapor-laravel-app
environments:
    production:
        subnets:
            - subnet-08aczain4man8bba7
            - subnet-08acr4nd0mbba7
        security-groups:
            - sg-0cr4and0m45b7e0
```

## Deleting Environments

Environments may be deleted via the Vapor UI or using the `env:delete` CLI command:

```bash
vapor env:delete testing
```
