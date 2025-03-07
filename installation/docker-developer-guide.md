# Docker Developer's Guide

This guide is for developers and newcomers to help them debug and explore Docker.

This page describes how to test and debug your changes once you have set up the project, Product Opener with Docker using [dev environment quick start guide](./dev-environment-quick-start-guide.md).

## Checking logs

### Tail Docker Compose logs

```
make log
```

### Tail other logs

```
make tail
```
It will `tail -f` all the files present in the `logs/` directory:

* `apache2/error.log`
* `apache2/log4perl.log`
* `apache2/modperl_error.log`
* `apache2/other_vhosts_access.log`
* `nginx/access.log`
* `nginx/error.log`

You can also simply run:
```
tail -f <FILEPATH>
```
to check a specific log.


## Opening a shell in a Docker container

Run the following to open a bash shell within the `backend` container:

```
docker-compose exec backend bash
```

You should see `root@<CONTAINER_ID>:/#` (opened root shell): you are now within the Docker container and can begin typing some commands !

### Checking permissions

Navigate to the directory the specific directory and run

```
ls -lrt
```
It will list all the directories and their permission status.

### Creating directory

Navigate to your specific directory using `cd` command and run

```
mkdir directory-name
```

### Running minion jobs

[Minion](https://docs.mojolicious.org/Minion) is a high-performance job queue for Perl. [Minion](https://docs.mojolicious.org/Minion) is used in [openfoodfacts-server](https://github.com/openfoodfacts/openfoodfacts-server) for time-consuming import and export tasks. These tasks are processed and queued using the minion jobs queue. Therefore, they are called minion jobs.

Go to `/opt/product-opener/scripts` and run

```
./minion_producers.pl minion job
```

The above command will show the status of minion jobs. Run the following command to launch the minion jobs.

```
./minion_producers.pl minion worker -m production -q pro.openfoodfacts.org
```

### Restarting Apache

Sometimes restarting the whole `backend` container is overkill, and you can just
restart `Apache` from inside the container:

```
apache2ctl -k restart
```

### Exiting the container

Use `exit` to exit the container.

## Making code changes

In the dev environment, any code change to the local directory will be written 
on the container. That said, some code changes require a restart of the `backend` 
container, or rebuilding the NPM assets.

### Manual reload

To restart the `backend` container after a config (`docker-compose.yml`, `docker/`, `.env`):
```
make restart
```

To restart `Apache` within the backend container after a code change (`lib/`):
```
make restart_apache
```

**Note:** restart is not necessary when making changes in the `cgi/` directory.

To rebuild frontend assets after an asset change (`html/` folder):
```
make up
```

### Live reload
To automate the live reload on code changes, you can install the Python package `when-changed`:
```
pip3 install when-changed
when-changed -r docker/ docker-compose.yml .env -c "make restart"                                         # restart backend container on compose changes
when-changed -r lib/ -r docker/ docker-compose.yml -c "make restart_apache"                               # restart Apache on code changes
when-changed -r html/ -r css/ -r scss/ -r icons/ Dockerfile Dockerfile.frontend package.json -c "make up" # rebuild containers on asset or Dockerfile changes
```

An alternative to `when-changed` is `inotifywait`.


## Run queries on MongoDB database
```
docker-compose exec mongodb mongo
```
The above command will open a MongoDB shell, allowing you to use all the `mongo` 
commands to interact with the database:

```
show dbs
use off
db.products.count()
db.products.find({_id: "5053990155354"})
db.products.deleteOne({_id: "5053990155354"})
```

See the [`mongo` shell docs](https://docs.mongodb.com/manual/reference/mongo-shell/) for more commands.

## Adding environment variables

If you need some value to be configurable, it is best to set is as an environment variable.

To add a new environment variable `TEST`:

* In `.env` file, add `TEST=test_val` [local].
* In `.github/workflows/container-deploy.yml`, add `echo "TEST=${{ secrets.TEST }}" >> .env` to the "Set environment variables" build step [remote]. Also add the corresponding GitHub secret `TEST=test_val`.
* In `docker-compose.yml` file, add it under the `backend` > `environment` section.
* In `conf/apache.conf` file, add `PerlPassEnv TEST`.
* In `lib/Config2.pm`, add `$test = $ENV{TEST};`. Also add `$test` to the `EXPORT_OK` list at the top of the file to avoid a compilation error.

The call stack goes like this:

`make up` > `docker-compose` > loads `.env` > pass env variables to the `backend` container > pass to `mod_perl` > initialized in `Config2.pm`.

## Managing multiple deployments

To juggle between multiple local deployments (e.g: to run different flavors of Open Food Facts on the same host), you will need:

* Multiple `.env` files (one per deployment), such as:

  * `.env.off` : configuration for Open Food Facts dev env.
  * `.env.off-pro` : configuration for Open Food Facts Producer's Platform dev env.
  * `.env.obf`: configuration for Open Beauty Facts dev env.
  * `.env.opff`: configuration for Open Ped Food Facts dev env.


* `COMPOSE_PROJECT_NAME` set to different values in each `.env` file, so that container names across deployments are unique.

* `FRONTEND_PORT` set to different values in each `.env` file, so that frontend containers don't port-conflict with each other.

To switch between configurations, set `ENV_FILE` before running `make` commands:

```
ENV_FILE=.env.off-pro make up # starts the OFF Producer's Platform containers.
ENV_FILE=.env.obf     make up # starts the OBF containers.
ENV_FILE=.env.opff    make up # starts the OPFF containers.
```

or export it to keep it for a while:
```
export ENV_FILE=.env.off # going to work on OFF for a while
make up
make restart
make down
make log
```

A good strategy is to have multiple terminals open, one for each deployment:

* `off` [Terminal 1]:
  ```
  export ENV_FILE=.env.off
  make up
  ```

* `off-pro` [Terminal 2]:
  ```
  export ENV_FILE=.env.off-pro
  make up
  ```

* `obf` [Terminal 3]:
  ```
  export ENV_FILE=.env.obf
  make up
  ```

* `opff` [Terminal 3]:
  ```
  export ENV_FILE=.env.opff
  make up
  ```

**Note:** the above case of 4 deployments is ***a bit ambitious***, since ProductOpener's `backend` container takes about ~6GB of RAM to run, meaning that the above 4 deployments would require a total of 24GB of RAM available.
