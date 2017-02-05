# How does fabalicious work in two sentences

Fabalicious uses a configuration file with a list of hosts and `ssh` and optionally tools like `composer`, `drush`, `git`, `docker` or custom scripts to run common tasks on remote machines. It is slightly biased to drupal-projects but it works for a lot of other types of projects.

Fabalicious is using [fabric](http://www.fabfile.org) to run tasks on remote machines. The configuration-file contains a list of hosts to work on. Some common tasks are:

 * deploying new code to a remote installation
 * reset a remote installation to its defaults.
 * backup/ restore data
 * copy data from one installation to another
 * optionally work with our docker-based development-stack [multibasebox](https://github.com/factorial-io/multibasebox)

# Running fabalicious

To execute a task with the help of fabalicious, just

```shell
cd <your-project-folder>
fab config:<your-config-key> <task>
```

This will read your fabfile.yaml, look for `<your-config-key>` in the host-section and run the task <task>

# Tasks

## Some Background

Fabalicious provides a set of so-called methods which implement all listed functionality. The following methods are available:

* git
* ssh
* drush
* composer
* files
* docker
* drupalconsole
* slack
* platform

You declare your needs in the fabfile.yaml with the key `needs`, e.g.

```yaml
needs:
  - git
  - ssh
  - drush
  - files
```

Have a look at the file-format documentation for more info.

## List of available tasks


You can get a list of available commands with

```shell
fab --list
```

## config

```shell
fab config:<your-config>
```

This is one of the most fundamental commands fabalicious provides. This will lookup `<your-config>` in the `hosts`-section of your `fabfile.yaml` and feed the data to `fabric` so it can connect to the host.

## list

```shell
fab list
```

This task will list all your hosts defined in your `hosts`-section of your `fabfile.yaml`.

## about

```shell
fab config:<your-config> about
```

will display the configuration of host `<your-config>`.

## blueprint

```shell
fab config:<your-config> blueprint:<branch-name>
fab blueprint:<branch-name>,configName=<config-name>
fab blueprint:<branch-name>,configNmae=<config-name>,output=True
```

`blueprint` will try to load a blueprint-template from the fabfile.yaml and apply the input given as `<branch-name>` to the template. This is helpful if you want to create/ use a new configuration which has some dynamic parts like the name of the database, the name of the docker-container, etc.

The task will look first in the host-config for the property `blueprint`, afterwards in the dockerHost-configuration `<config-name>` and eventually in the global namespace. If you wnat to print the generated configuration as yaml, then add `,output=true` to the command. If not, the generated configuration is used as the current configuration, that means, you can run other tasks against the generated configuration.

**Available replacement-patterns** and what they do. Input is `feature/XY-123-my_Branch-name`, the project-name is `Example project`:

- `%slug.with-hyphens.without-feature%` => `xy-123-my-branch-name`
- `%slug.with-hyphens%` => `feature-xy-123-my-branch-name`
- `%project-slug.with-hypens%` => `example-project`
- `%slug%` => `featurexy123mybranchname`
- `%project-slug%` => `exampleproject`
- `%project-identifier%` => `Example project`
- `%identifier%` => `feature/XY-123-my_Branch-name`
- `%slug.without-feature%` => `xy123mybranchname`

Here's an example blueprint:

```yaml
blueprint:
  inheritsFrom: http://some.host/data.yaml
  configName: '%project-slug%-%slug.with-hyphens.without-feature%.some.host.tld'
  branch: '%identifier%'
  database:
    name: '%slug.without-feature%_mysql'
  docker:
    projectFolder: '%project-slug%--%slug.with-hyphens.without-feature%'
    vhost: '%project-slug%-%slug.without-feature%.some.host.tld'
    name: '%project-slug%%slug.without-feature%_web_1'
```

And the output of `fab blueprint:feature/XY-123-my_Branch-name,configNamy=<config-name>,output=true` is

```yaml
hosts:
  phbackend-xy-123-my-branch-name.some.host.tld:
    branch: feature/XY-123-my_Branch-name
    configName: phbackend-xy-123-my-branch-name.some.host.tld
    database:
      name: xy123mybranchname_mysql
    docker:
      name: phbackendxy123mybranchname_web_1
      projectFolder: phbackend--xy-123-my-branch-name
      vhost: phbackend-xy123mybranchname.some.host.tld
    inheritsFrom: http://some.host/data.yaml
```


## doctor

The `doctor`-task will try to establish all needed ssh-connections and tunnels and give feedback if anything fails. This should be the task you run if you have any problems connecting to a remote instance.

```shell
fab config:<your-config> doctor
fab config:<your-config> doctor:remote=<your-remote-config>
```
Running the doctor-task without an argument, will test the connectivity to the configuration `<your-cofig>`. If you provide a remote configuration with `:remote=<your-remote-config>` the doctor command will create and test any necessary tunnels to test the connections betwenn `<your-config>` and `<your-remote-config>`. Might be handy if the task `copyFrom` fails.


## getProperty

```shell
fab config:<your-config> getProperty:<name-of-property>
```

This will print the property-value to the console. Suitable if you want to use fabalicious from within other scripts.

**Examples**
* `fab config:mbb getProperty:host` will print the hostname of configuration `mbb`.
* `fab config:mbb getProperty:docker/tag` will print the tag of the docker-configuration of `mbb`.


## version

```shell
fab config:<your-config> version
```

This command will display the installed version of the code on the installation `<your-config>`.

**Available methods**:
* `git`. The task will get the installed version via `git describe`, so if you tag your source properly (hint git flow), you'll get a nice version-number.

**Configuration:**
* your host-configuration needs a `branch`-key stating the branch to deploy.

## deploy

```shell
fab config:<your-config> deploy
fab config:<your-config> deploy:<branch-to-deploy>
```

This task will deploy the latest code to the given installation. If the installation-type is not `dev` or `test` the `backupDB`-task is run before the deployment starts. If `<branch-to-deploy>` is stated the specific branch gets deployed.

After a successfull deployment the `reset`-taks will be run.

**Available methods:**
* `git` will deploy to the latest commit for the given branch defined in the host-configuration. Submodules will be synced, and updated.
* `platform` will push the current branch to the `platform` remote, which will start the deployment-process on platform.sh

**Configuration:**
* your host-configuration needs a `branch`-key stating the branch to deploy.

## reset

```shell
fab config:<your-config> reset
```

This task will reset your installation

**Available methods:**
* `composer` will run `composer install` to update any dependencies before doing the reset
* `drush` will
  * set the site-uuid from fabfile.yaml (drupal 8)
  * revert features (drupal 7) / import the configuration `staging` (drupal 8),
  * run update-hooks
  * enable a deployment-module if any stated in the fabfile.yaml
  * and does a cache-clear.
  * if your host-type is `dev` and `withPasswordReset` is not false, the password gets reset to admin/admin


**Configuration:**
* your host-configuration needs a `branch`-key stating the branch to deploy.
* your configuration needs a `uuid`-entry, this is the site uuid (drupal 8). You can get the site-uuid via `drush cget system.site`
* you can customize which configuration to import with the `configurationManagement`-setting inside your host- or global-setting.

**Examples:**
* `fab config:mbb reset:withPasswordReset=0` will reset the installation and will not reset the password.


## backup

```shell
fab config:<your-config> backup
```

This command will backup your files and database into the specified `backup`-directory. The file-names will include configuration-name, a timestamp and the git-SHA1. Every backup can be referenced by its filename (w/o extension) or, when git is abailable via the git-commit-hash.

**Available methods:**
* `git` will prepend the file-names with a hash of the current revision.
* `files` will tar all files in the `filesFolder` and save it into the `backupFolder`
* `drush` will dump the databases and save it to the `backupFolder`

**Configuration:**
* your host-configuration will need a `backupFolder` and a `filesFolder`


## backupDB

```shell
fab config:<your-config> backupDB
```

This command will backup only the database. See the task `backup` for more info.


## listBackups

```shell
fab config:<your-config> listBackups
```

This command will print all available backups to the console.


## restore

```shell
fab config:<your-config> restore:<commit-hash|file-name>
```

This will restore a backup-set. A backup-set consists typically of a database-dump and a gzupped-file-archive. You can a list of candidates via `fab config:<config> listBackups`

**Available methods**
* `git` git will checkout the given hash encoded in the filename.
* `files` all files will be restored. An existing files-folder will be renamed for safety reasons.
* `drush` will import the database-dump.


## getBackup

```shell
fab config:<config> getBackup:<commit-hash|file-name>
```

This command will copy a remote backup-set to your local computer into the current working-directory.

**See also:**
* restore
* backup


## copyFrom

```shell
fab config:<dest-config> copyFrom:<source-config>
```

This task will copy all files via rsync from `source-config`to `dest-config` and will dump the database from `source-config` and restore it to `dest-config`. After that the `reset`-task gets executed. This is the ideal task to copy a complete installation from one host to another.

**Available methods**
* `ssh` will create all necessary tunnels to access the hosts.
* `files` will rsync all new and changed files from source to dest
* `drush` will dump the database and restore it on the dest-host.


## copyDBFrom

```shell
fab config:<dest-config> copyDBFrom:<source-config>
```

Basically the same as the `copyFrom`-task, but only the database gets copied.


## copyFilesFrom

```shell
fab config:<dest-config> copyFileFrom:<source-config>
```

Basically the same as the `copyFrom`-task, but only the new and updated files get copied.


## drush

```shell
fab config:<config> drush:<drush-command>
```

This task will execute the `drush-command` on the remote host specified in <config>. Please note, that you'll have to quote the drush-command when it contains spaces. Signs should be excaped, so python does not interpret them.

**Available methods**
* Only available for the `drush`-method

**Examples**
* `fab config:staging drush:"cc all"`
* `fab config:local drush:fra`


## drupalconsole

This task will execute a drupal-console task on the remote host. Please note, that you'll have to quote the command when it contains spaces. There's a special command to install the drupal-console on the host: `fab config:<config> drupalconsole:install`

**Available methods**
* Only available for the `drupalconsole`-method

**Examples**
* `fab config:local drupalconsole:cache:rebuild`
* `fab config:local drupalconsole:"generate:module --module helloworld"`


## getFile

```shell
fab config:<config> getFile:<path-to-remote-file>
```

Copy a remote file to the current working directory of your current machine.


## putFile

```shell
fab config:<config> putFile:<path-to-local-file>
```

Copy a local file to the tmp-folder of a remote machine.

**Configuration**
* this command will use the `tmpFolder`-host-setting for the destination directory.


## getSQLDump

```shell
fab config:<config> getSQLDump
```

Get a current dump of the remote database and copy it to the local machine into the current working directory.

**Available methods**
* currently only implemented for the `drush`-method


## restoreSQLFromFile

```shell
fab config:<config> restoreSQLFromFile:<path-to-local-sql-dump>
```

This command will copy the dump-file `path-to-local-sql-dump` to the remote machine and import it into the database.

**Available methods**
* currently only implemented for the `drush`-method


## script

```shell
fab config:<config> script:<script-name>
```

This command will run custom scripts on a remote machine. You can declare scripts globally or per host. If the `script-name` can't be found in the fabfile.yaml you'll get a list of all available scripts.

Additional arguments get passed to the script. You'll have to use the python-syntax to feed additional arguments to the script. See the examples.

**Examples**
* `fab config:mbb script`. List all available scripts for configuration `mbb`
* `fab config:mbb script:behat` Run the `behat`-script
* `fab config:mbb script:behat,--name="Login feature",--format=pretty` Run the behat-test, apply `--name` and `--format` parameters to the script

The `script`-command is rather powerful, have a read about it in the extra section.


## docker

```shell
fab config:<config> docker:<docker-task>
```

The docker command is suitable for orchestrating and administering remote instances of docker-containers. The basic setup is that your host-configuration has a `docker`-section, which contains a `configuration`-key. The `dockerHosts`-section of your fabfile.yaml has a list of tasks which are executed on the "parent-host" of the configuration. Please have a look at the docker-section for more information.

Most of the time the docker-container do not have a public or known ip-address. Fabalicious tries to find out the ip-address of a given instance and use that for communicating with its services.

There are three implicit tasks available:

### copySSHKeys

```shell
fab config:mbb docker:copySSHKeys
```

This will copy the ssh-keys into the docker-instance. You'll need to provide the paths to the files via the three configurations:
* `dockerKeyFile`, the path to the private ssh-key to use.
* `dockerAuthorizedKeyFile`, the path to the file for `authoried_keys`
* `dockerKnownHostsFile`, the path to the file for `known_hosts`

As docker-container do not have any state, this task is used to copy any necessary ssh-configuration into the docker-container, so communication per ssh does not need any passwords.

### startRemoteAccess

```shell
fab config:<config> docker:startRemoteAccess
fab config:<config> docker:startRemoteAccess,port=<port>,publicPort=<public-port>
```

This docker-task will run a ssh-command to forward a local port to a port inside the docker-container. It starts a new ssh-session which will do the forwarding. When finished, type `exit`.

**Examples**
* `fab config:mbb docker:startRemoteAccess` will forward `localhost:8888` to port `80` of the docker-container
* `fab config:mbb docker:startRemoteAccess,port=3306,publicPort=33060` will forward `localhost:33060`to port `3306` of the docker-container

### waitForServices

This task will try to establish a ssh-connection into the docker-container and if the connection succeeds, waits for `supervisorctl status` to return success. This is useful in scripts to wait for any services that need some time to start up. Obviously this task depends on `supervisorctl`.


# the structure of the configuration file

## Overview

The configuration is fetched from the file `fabfile.yaml` and should have the followin structure:

```yaml
name: <the project name>

needs:
  - list of methods

requires: 2.0

dockerHosts:
  docker1:
    ...

hosts:
  host1:
    ...
```

Here's the documentation of the supported and used keys:

### name

The name of the project, it's only used for output.

### needs

List here all needed methods for that type of project. Available methods are:
  * `git` for deployments via git
  * `ssh`
  * `drush7` for support of drupal-7 installations
  * `drush8` for support fo drupal 8 installations
  * `files`
  * `slack` for slack-notifications
  * `docker` for docker-support
  * `composer` for composer support
  * `drupalconsole` for drupal-concole support

**Example for drupal 7**

```yaml
needs:
  - ssh
  - git
  - drush7
  - files
```

**Example for drupal 8 composer based and dockerized**

```yaml
needs:
  - ssh
  - git
  - drush8
  - composer
  - docker
  - files
```


### requires

The file-format of fabalicious changed over time. Set this to the lowest version of fabalicious which can handle the file. Should bei `2.0`

### hosts

Hosts is a list of host-definitions which contain all needed data to connect to a remote host. Here's an example

```yaml
hosts:
  exampleHost:
    host: example.host.tld
    user: example_user
    port: 2233
    password: optionalPassword
    type: dev
    rootFolder: /var/www/public
    gitRootFolder: /var/www
    siteFolder: /sites/default
    filesFolder: /sites/default/files
    backupFolder: /var/www/backups
    supportsInstalls: true|false
    supportsCopyFrom: true|false
    type: dev
    branch: develop
    docker:
      ...
    database:
      ...
    scripts:
      ...
    sshTunnel:
      ..

```

You can get all host-information including the default values using the fabalicious command `about`:

```shell
fab config:staging about
```

This will print all host configuration for the host `staging`.

Here are all possible keys documented:

* `host`, `user`, `port` and optionally `password` is used to connect via SSH to the remote machine. Please make sure SSH key forwarding is enabled on your installation. `password` should only used as an exception.
* `type` defines the type of installation. Currently there are four types available:
    * `dev` for dev-installations, they won't backup the databases on deployment
    * `test` for test-installations, similar than `dev`, no backups on deployments
    * `stage` for staging-installations.
    * `live` for live-installations. Some tasks can not be run on live-installations as `install` or as a target for `copyFrom`
    The main use-case is to run different scripts per type, see the `common`-section.
* `branch` the name of the branch to use for deployments, they get ususally checked out and pulled from origin. `gitRootFolder` should be the base-folder, where the local git-repository is. (If not explicitely set, fabalicious uses the `rootFolder`)
* `rootFolder`  the web-root-folder of the installation, typically exposed to the public.
* `backupFolder` the folder, where fabalicious should store its backups into
* `runLocally` if set to true, all commands are run on the local host, not on a remote host. Good for local development on linux or tools like MAMP.
* `siteFolder` is a drupal-specific folder, where the settings.php resides for the given installation. This allows to interact with multisites etc.
* `filesFolder` the path to the files-folder, where user-assets get stored and which should be backed up by the `files`-method
* `tmpFolder` name of tmp-folder, defaults to `/tmp`
* `supportsBackups` defaults to true, set to false, if backups are not supported
* `supportsZippedBackups` defaults to true. Set to false, if database-dumps shouldn't be zipped.
* `supportsInstalls` defaults to false, if set to true, the `install`-task will run on that host.
* `supportsCopyFrom` defaults to false, if set to true, the host can be used as target for `copyFrom`
* `ignoreSubmodules` defaults to true, set to false, if you don't want to update a projects' submodule on deploy.
* `configurationManagement`, an array of configuration-labels to import on `reset`, defaults to `['staging']`. You can add command arguments for drush, e.g. `['staging', 'dev --partial']`
* `disableKnownHosts`, `useShell` and `usePty` see section `other`
* `database` the database-credentials the `install`-tasks uses when installing a new installation.
    * `name` the database name
    * `host` the database host
    * `user` the database user
    * `pass` the password for the database user
    * `prefix` the optional table-prefix to use
* `sshTunnel` Fabalicious supports SSH-Tunnels, that means it can log in into another machine and forward the access to the real host. This is handy for dockerized installations, where the ssh-port of the docker-instance is not public. `sshTunnel` needs the following informations
    * `bridgeHost`: the host acting as a bridge.
    * `bridgeUser`: the ssh-user on the bridge-host
    * `bridgePort`: the port to connect to on the bridge-host
    * `localPort`: the local port which gets forwarded to the `destPort`. If `localPort` is omitted, the ssh-port of the host-configuration is used. If the host-configuration does not have a port-property a random port is used.
    * `destHost`: the destination host to forward to
    * `destHostFromDockerContainer`: if set, the docker's Ip address is used for destHost. This is automatically set when using a `docker`-configuration, see there.
    * `destPort`: the destination port to forward to
* `docker` for all docker-relevant configuration. `configuration` and `name` are the only required keys, all other are optional and used by the docker-tasks.
    * `configuration` should contain the key of the dockerHost-configuration in `dockerHosts`
    * `name` contains the name of the docker-container. This is needed to get the IP-address of the particular docker-container when using ssh-tunnels (see above).



### dockerHosts

`dockerHosts` is similar structured as the `hosts`-entry. It's a keyed lists of hosts containing all necessary information to create a ssh-connection to the host, controlling the docker-instances, and a list of tasks, the user might call via the `docker`-command. See the `docker`-entry for a more birds-eye-view of the concepts.

Here's an example `dockerHosts`-entry:

```yaml
dockerHosts:
  mbb:
    runLocally: false
    host: multibasebox.dev
    user: vagrant
    password: vagrant
    port: 22
    rootFolder: /vagrant
    environment:
      VHOST: %host.host%
      WEBROOT: %host.rootFolder%
    tasks:
      logs:
        - docker logs %host.docker.name%
```

Here's a list of all possible entries of a dockerHosts-entry:

* `runLocally`: if this is set to `true`, all docker-scripts are run locally, and not on a remote host.
* `host`, `user`, `port` and `password`: all needed information to start a ssh-connection to that host. These settings are only respected, if `runLocally` is set to `false`. `port` and `password` are optional.
* `environment` a keyed list of environment-variables to set, when running one of the tasks. The replacement-patterns of `scripts` are supported, see there for more information.
* `tasks` a keyed list of commands to run for a given docker-subtask (similar to `scripts`). Note: these commands are running on the docker-host, not on the host. All replacement-patterns do work, and you can call even other tasks via `execute(<task>, <subtask>)` e.g. `execute(docker, stop)` See the `scripts`-section for more info.

You can use `inheritsFrom` to base your configuration on an existing one. You can add any configuration you may need and reference to that information from within your tasks via the replacement-pattern `%dockerHost.keyName%` e.g. `%dockerHost.host%`.

You can reference a specific docker-host-configuration from your host-configuration via

```yaml
hosts:
  test:
    docker:
      configuration: mbb
```

### common

common contains a list of commands, keyed by task and type which gets executed when the task is executed.

Example:
```yaml
common:
  reset:
    dev:
      - echo "running reset on a dev-instance"
    stage:
      - echo "running reset on a stage-instance"
    prod:
      - echo "running reset on a prod-instance"
  deployPrepare:
    dev:
      - echo "preparing deploy on a dev instance"
  deploy:
    dev:
      - echo "deploying on a dev instance"
  deployFinished:
    dev:
      - echo "finished deployment on a dev instance"
```

The first key is the task-name (`reset`, `deploy`, ...), the second key is the type of the installation (`dev`, `stage`, `prod`, `test`). Every task is prepended by a prepare-stage and appended by a finished-stage, so you can call scripts before and after an actual task. You can even run other scripts via the `execute`-command, see the `scripts`-section.

### scripts

A keyed list of available scripts. This scripts may be defined globally (on the root level) or on a per host-level. The key is the name of the script and can be executed via

```shell
fab config:<configuration> script:<key>
```

A script consists of an array of commands which gets executed sequentially.

An example:

```yaml
scripts:
  test:
    - echo "Running script test"
  test2:
    - echo "Running script test2 on %host.config_name%
    - execute(script, test)
```

Scripts can be defined on a global level, but also on a per host-level.

You can declare default-values for arguments via a slightly modified syntax:

```yaml
scripts:
  defaultArgumentTest:
    defaults:
      name: Bob
    script:
      - echo "Hello %arguments.name%"
```

Running the script via `fab config:mbb script:defaultArgumentTest,name="Julia"` will show `Hello Julia`. Running `fab config:mbb script:defaultArgumentTest` will show `Hello Bob`.

For more information see the main scripts section below.

### other

* `deploymentModule` name of the deployment-module the drush-method enables when doing a deploy
* `usePty` defaults to true, set it to false when you can't connect to specific hosts.
* `useShell` defaults to true, set it to false, when you can't connect to specific hosts.
* `disableKnownHosts` defaults to false, set it too true, if you trust every host
* `gitOptions` a keyed list of options to apply to a git command. Currently only pull is supported. If your git-version does not support `--rebase` you can disable it via an empty array: `pull: []`
* `sqlSkipTables` a list of table-names drush should omit when doing a backup.
* `configurationManagement` a list of configuration-labels to import on `reset`. This defaults to `['staging']` and may be overridden on a per-host basis. You can add command arguments to the the configuration label.

Example:
```yaml
deploymentModule: my_deployment_module
usePty: false
useShell: false
gitOptions:
  pull:
    - --rebase
    - --quiet
sqlSkipTables:
  - cache
  - watchdog
  - session
configurationManagement:
   - staging
   - dev -- partial
```


## Inheritance

Sometimes it make sense to extend an existing configuration or to include configuration from other places from the file-system or from remote locations. There's a special key `inheritsFrom` which will include the yaml found at the location and merge it with the data. This is supported for entries in `hosts` and `dockerHosts` and for the fabfile itself.

If a `host`, a `dockerHost` or the fabfile itself has the key `inheritsFrom`, then the given key is used as a base-configuration. Here's a simple example:

```yaml
hosts:
  default:
    port: 22
    host: localhost
    user: default
  example1:
    inheritsFrom: default
    port: 23
  example2:
    inheritsFrom: example1
    user: example2
```

`example1` will store the merged configuration from `default` with the configuration of `example1`. `example2` is a merge of all three configurations: `example2` with `example1` with `default`.

```yaml
hosts:
  example1:
    port: 23
    host: localhost
    user: default
  example2:
    port: 23
    host: localhost
    user: example2
```

You can even reference external files to inherit from:

```yaml
hosts:
  fileExample:
    inheritsFrom: ./path/to/config/file.yaml
  httpExapme:
    inheritsFrom: http://my.tld/path/to/config_file.yaml
```

This mechanism works also for the fabfile.yaml / index.yaml itself, and is not limited to one entry:

```yaml
name: test fabfile

inheritsFrom:
  - ./mbb.yaml
  - ./drupal.yaml
```



# Scripts

Scripts are a powerful concept of fabalicious. There are a lot of places where scripts can be called. The `common`-section defines common scripts to be run for specific task/installation-type-configurations, docker-tasks are also scripts which you can execute via the docker-command. And you can even script fabalicious tasks and create meta-tasks.

A script is basically a list of commands which get executed via shell on a remote machine. To stay independent of the host where the script is executed, fabalicious parsed the script before executing it and replaces given variables with their counterpart in the yams file.

## Replacement-patterns

Replacement-Patterns are specific strings enclosed in `%`s, e.g. `%host.port%`, `%dockerHost.rootFolder% or `%arguments.name%.

Here's a simple example;

```yaml
script:
  test:
    - echo "I am running on %host.config_name%"
```

Calling this script via

```shell
fab config:mbb script:test
```

will show `I am running on mbb`.

* The host-configuration gets exposes via the `host.`-prefix, so `port` maps to `%host.port%`, etc.
* The dockerHost-configuration gets exposed via the `dockerHost`-prefix, so `rootFolder` maps to `%dockerHost.rootFolder%`
* The global configuration of the yams-file gets exposed to the `settings`-prefix, so `uuid` gets mapped to `%settings.uuid%
* Optional arguments to the `script`-taks get the `argument`-prefix, e.g. `%arguments.name%`. You can get all arguments via `%arguments.combined%`.
* You can access hierarchical information via the dot-operator, e.g. `%host.database.name%`

If fabalicious detects a pattern it can't replace it will abort the execution of the script and displays a list of available replacement-patterns.

## Internal commands

There are currently 3 internal commands. These commands control the flow inside fabalicious:

* `fail_on_error(1|0)` If fail_on_error is set to one, fabalicious will exit if one of the script commands returns a non-zero return-code. When using `fail_on_error(0)` only a warning is displayed, the script will continue.
* `execute(task, subtask, arguments)` execute a fabalicious task. For example you can run a deployment from a script via `execute(deploy)` or stop a docker-container from a script via `execute(docker, stop)`
* `fail_on_missing_directory(directory, message)` will print message `message` if the directory `directory` does not exist.

## Task-related scripts

You can add scripts to the `common`-section, which will called for any host. You can differentiate by task-name and host-type, e.g. create a script which gets called for the task `deploy` and type `dev`.

You can even run scripts before or after a task is executed. Append the task with `Prepare` or `Finished`.

You can even run scripts for specific tasks and hosts. Just add your script with the task-name as its key.

```yaml
host:
  test:
    deployPrepare:
      - echo "Preparing deploy for test"
    deploy:
      - echo "Deploying on test"
    deployFinished:
      - echo "Deployment finished for test"
```

These scripts in the above examples gets executed only for the host `test` and task `deploy`.

## Examples

A rather complex example scripting fabalicious.

```yaml
scripts:
  runTests:
    defaults:
      branch: develop
    script:
      - execute(docker, start)
      - execute(docker, waitForServices)
      - execute(deploy, %arguments.branch%)
      - execute(script, behatInstall)
      - execute(script, behat, --profile=ci --format=junit --format=progress)
      - execute(getFile, /var/www/_tools/behat/build/behat/default.xml, ./_tools/behat)
      - execute(docker, stop)
```

This script will
* start the docker-container,
* wait for it,
* deploys the given branch,
* run a script which will install behat,
* run behat with some custom arguments,
* gets the result-file and copy it to a location,
* and finally stops the container.



# Docker integration

TODO

# Local overrides

`fabfile.local.yaml` is used to override parts of your fabfile-configuration. If you run a fab-command the code will try to find the `fabfile.local.yaml` up to three folder levels up and merge the data with your fabfile.yaml.

A small example:

```
fabfile.local.yaml
+ project
  fabfile.yaml
```

Contents fo fabfile.yaml
```yaml
hosts:
  local:
    host: multibasebox.dev
    port: 22
    [...]
```

Contents of fabfile.local.yaml:
```yaml
hosts:
  local:
    host: localhost
    port: 2222
```

This will override the `host` and `port` settings of the `local`-configuration. With this technique you can alter an existing fabfile.yaml with local overrides. (In this example,  `host=localhost` and `port=2222`