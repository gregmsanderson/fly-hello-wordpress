# fly-hello-wordpress

A simple example of running WordPress on [Fly.io](https://fly.io). It uses the official [WordPress Docker image](https://hub.docker.com/_/wordpress) which currently uses Apache & PHP 7.4.

## Run it locally

At a minimum you need to provide environment variables for a MySQL database: its name, hostname, username and password. That database must already exist (WordPress will not create it):

```
docker build --tag wordpress-demo .

docker run -p 8080:80 \
-e WORDPRESS_DB_HOST='hostname-here' \
-e WORDPRESS_DB_USER='username-here' \
-e WORDPRESS_DB_PASSWORD='password-here' \
-e WORDPRESS_DB_NAME='database-name-here' \
wordpress-demo
```

If you then visit `http://localhost:8080/` you should see the WordPress installation page. Pick your language, provide some user details and it will then install into your provided database.

## Environment variables

The image supports more environment variables. For example:

```
docker run -p 8080:80 \
-e WORDPRESS_DEBUG=1 \
-e WORDPRESS_DB_HOST='hostname-here' \
-e WORDPRESS_DB_USER='user-name-here' \
-e WORDPRESS_DB_PASSWORD='password-here' \
-e WORDPRESS_DB_NAME='database-name-here' \
-e WORDPRESS_TABLE_PREFIX='prefix-here' \
-e WORDPRESS_CONFIG_EXTRA='define("MYSQL_CLIENT_FLAGS", MYSQLI_CLIENT_SSL);' \
-e WORDPRESS_AUTH_KEY='get-from-wordpress-generator' \
-e WORDPRESS_SECURE_AUTH_KEY='get-from-wordpress-generator)' \
-e WORDPRESS_LOGGED_IN_KEY='get-from-wordpress-generator' \
-e WORDPRESS_NONCE_KEY='get-from-wordpress-generator' \
-e WORDPRESS_AUTH_SALT='get-from-wordpress-generator' \
-e WORDPRESS_SECURE_AUTH_SALT='get-from-wordpress-generator' \
-e WORDPRESS_LOGGED_IN_SALT='get-from-wordpress-generator' \
-e WORDPRESS_NONCE_SALT='get-from-wordpress-generator' \
wordpress-demo
```

The `WORDPRESS_DEBUG` can be used _if_ you experience any problems and want to see more debugging information.

The `WORDPRESS_TABLE_PREFIX` is usually `wp_` but can be any string. That's useful if your database contains other tables.

The `WORDPRESS_CONFIG_EXTRA` is used to add additional values to the config file. Here we are using that to specify SSL should be used to connect to a database. That is needed if you want to connect to a [PlanetScale](https://planetscale.com) database.

The `WORDPRESS_AUTH_KEY` (and onwards) are security keys/salts. If you change them, all users will be forced to login again. We recommend using their official generator: [https://api.wordpress.org/secret-key/1.1/salt](https://api.wordpress.org/secret-key/1.1/salt). As shown above, the variables it generates need to be prefixed with `WORDPRESS_` for the image to recognise them.

## Deploy to Fly

If you haven't already done so, [install the Fly CLI](https://fly.io/docs/getting-started/installing-flyctl/) and then [log in to Fly](https://fly.io/docs/getting-started/log-in-to-fly/).

We have provided a base `fly.toml` file which tells Fly how to configure the app. Make sure to change the name at the top to what you plan to call _your_ app, for example **your-app-name**. You can adjust the other values within it later.

Run `fly launch` from the application's directory.

The Fly CLI will spot the existing `fly.toml`:

```
An existing fly.toml file was found for your-app-name
? Would you like to copy its configuration to the new app? (y/N)
```

Type **y** (yes).

The CLI will then spot the `Dockerfile`:

```
Scanning source code
Detected a Dockerfile app
```

You'll be asked to give the app a name. Type in a name using lowercase characters and hyphens. For example **your-app-name**.

Proceed through the prompts, choosing an organization, a region, and then say **N** (no) when asked to set up a Postgresql database. Finally, when it says do you want to deploy now, say **N** (no).

Why not deploy right now? Well, we need some environment variables to be set. At a minimum we need to provide a MySQL database's name, hostname, username and password. We will provide those as secrets.

So next use the `fly secrets` command to set all of the environment variables not referenced in the `[env]` section of the `fly.toml`. For example your command may look like:

```
fly secrets set \
WORDPRESS_DB_HOST='hostname-here' \
WORDPRESS_DB_USER='user-name-here' \
WORDPRESS_DB_PASSWORD='password-here' \
WORDPRESS_DB_NAME='database-name-here' \
WORDPRESS_TABLE_PREFIX='prefix-here' \
WORDPRESS_AUTH_KEY='get-from-wordpress-generator' \
WORDPRESS_SECURE_AUTH_KEY='get-from-wordpress-generator)' \
WORDPRESS_LOGGED_IN_KEY='get-from-wordpress-generator' \
WORDPRESS_NONCE_KEY='get-from-wordpress-generator' \
WORDPRESS_AUTH_SALT='get-from-wordpress-generator' \
WORDPRESS_SECURE_AUTH_SALT='get-from-wordpress-generator' \
WORDPRESS_LOGGED_IN_SALT='get-from-wordpress-generator' \
WORDPRESS_NONCE_SALT='get-from-wordpress-generator'
```

That should be successful and say those secrets are staged for the first deployment.

Now you can go ahead and deploy the app. Run `fly deploy`.

You should see the build proceed, the image pushed to Fly, and it create a release:

```
==> Monitoring deployment

1 desired, 1 placed, 0 healthy, 0 unhealthy
```

Once complete type `fly open` to visit `https://your-app-name.fly.dev`. After a few seconds you should see the WordPress installation page. Choose your language and click _Continue_ to proceed. On the next page provide the details asked for, click _Install Wordpress_ and it will then add tables to your chosen database. It may take up to 30 seconds. You should then see its success message: _"WordPress has been installed. Thank you, and enjoy!"_.

## MySQL database

This guide assumes you already have a MySQL database and so does not cover creating one. You may want to [create a Fly app for a MySQL database](https://fly.io/docs/app-guides/mysql-on-fly/). Or use an external service such as Planetscale.

## Custom theme/plugins?

Please see the documentation for the [WordPress image](https://hub.docker.com/_/wordpress). The `.dockerignore` and `Dockerfile` would need adapting to copy those into the image.

## Persistent storage

This sample app does not include any persistent storage. Its file system is ephemeral and should be used for temporary data (such as caches). If you _do_ need data to persist (for example uploaded images), you should either upload it to an external store (such as AWS S3) or create a [volume](https://fly.io/docs/reference/volumes/).

## Custom PHP settings

If you want to use custom PHP settings (overriding the default ones the WordPress image uses) those should go in a `php.ini` file. Make sure your `.dockerignore` file does _not_ ignore that. In our case, we would add a line such as `!php.ini`. The `Dockerfile` then needs to reference it. So that would need to include a line such as `COPY php.ini $PHP_INI_DIR/conf.d/` to make sure it is copied to where WordPress expects it to be.

## Errors?

If you see any errors, the first place to check is the log. Run `fly logs` from within the application's directory. Do you see anything to indicate why?

If the TCP healthchecks fail and your vm is marked as unhealthy, make sure your `fly.toml` has the app listening on port 80 (internally). Many apps use 8080.

You can try temporarily setting `WORDPRESS_DEBUG` as `1` within your `fly.toml` (in the `[env]` section). That will show additional PHP debug data when you load a page. For example if it complains it can't connect to your database, that can reveal why not.

To check secrets are being correctly set/read, run `fly ssh console` to SSH in to the vm, then run e.g `echo $WORDPRESS_DB_HOST` to see the value of that environment variable. If a required one is missing or incorrect, WordPress won't be able to connect to your database.

## Misc.

To see what files a built Docker image contains locally, one way is to use [dive](https://github.com/wagoodman/dive) and then `dive [id]`. Press tab/spacebar to look at the file tree.

When deployed to Fly, you can SSH in to the vm using `fly ssh console` and then view the file system. For example `cd /var/www/html`, and `ls -l`. You should see the WordPress files (`index.php` etc).
