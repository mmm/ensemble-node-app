
# Ensemble formula to deploy a user-defined node.js app

This is an example 
[ensemble](http://ensemble.ubuntu.com)
formula to deploy a user-defined node app
directly from revision control.

This formula will be maintained with a general set of hooks
for various services that can be used with node apps
(like mongodb).


# Using this formula

First, edit `config.yaml` to add info about your app.

Then deploy some basic services

    $ ensemble deploy --repository ~/formulas node-app myapp
    $ ensemble deploy --repository ~/formulas mongodb
    $ ensemble deploy --repository ~/formulas haproxy

relate them

    $ ensemble add-relation mongodb myapp
    $ ensemble add-relation myapp haproxy

scale up your app

    $ for i in {1..10}; do
    $   ensemble add-unit myapp
    $ done

open it up to the outside world

    $ ensemble expose haproxy

Find the haproxy instance's public URL from 

    $ ensemble status

(or attach it to an elastic IP via the aws console)
and open it up in a browser.


## What the formula does

During the `install` hook,

- installs `node`/`npm`
- clones your node app from the repo specified in `app_repo`
- runs `npm` if your app contains `package.json`
- configures networking if your app contains `config/config.js`
- waits to be joined to a `mongodb` service

when related to a `mongodb` service, the formula

- configures db access if your app contains `config/config.js`
- starts your node app as a service


## Formula configuration

Configurable aspects of the formula are listed in `config.yaml`
and can be set by either editing the default values directly
in the yaml file or passing a `myapp.yaml` configuration
file during deployment

    $ ensemble deploy --repository ~/formulas --config ~/myapp.yaml node-app myapp

Some of these parameters are used directly by the formula,
and some are passed through to the node app using `config/config.js`.

## Application configuration

The formula looks for `config/config.js` in your app which
starts off looking something like this

    module.exports = config = {
      "name" : "mynodeapp"
      ,"listen_port" : 8000
      ,"mongo_host" : "localhost"
      ,"mongo_port" : 27017
    }


and gets modified with contextually correct configuration information during
either deployment (via the `install` hook) or relation to another service 
(`relation-changed` hook).

This config can be used from within
your application using snippets like

    ...
    var config = require('./config/config')
    ...
      new mongo.Server(config.mongo_host, config.mongo_port, {}),
    ...
    server.listen(config.listen_port);
    ...



## Network access

This formula does not open any public ports itself.
The intention is to relate it to a proxy service like
`haproxy`, which will in turn open port 80 to the outside world.
This allows for instant horizontal scalability.

If your node app is itself a proxy and you want it directly exposed,
this can easily be done by adding 

    open-port $app_port

to the bottom of the `install` hook, and then once your stack
is started, you expose

    $ ensemble expose myapp

it to the outside world.

By default, ensemble services within the same environment
can talk to each other on any port over
internal network interfaces.


# Making this work with your node.js app

This formula makes some strong assumptions
about the structure of the node application 
(`config/config.js`) that might not apply to your app.
Please treat this formula as a template that 
you can fork and modify to suit your needs.

The biggest difference between how the formula
behaves for different kind of apps is application
startup.  A simple application will want to start
upon install (startup code goes in the `install` hook),
whereas some applications will not want
to start up until a database has be associated
(startup code goes in the `db-relation-joined` hooks).


# Mirrored

This repo is mirrored between http://github.com/mmm/ensemble-node-app
and lp:principia/node-app 
(https://code.launchpad.net/~mark-mims/principia/oneiric/node-app/trunk)
