# Electrode Confippet [![NPM version][npm-image]][npm-url] [![Build Status][travis-image]][travis-url] [![Dependency Status][daviddm-image]][daviddm-url]

Confippet is a versatile, flexible utility for managing configurations of
Node.js applications. It's simple to get started, and can be customized and
extended to meet the needs of your app.

<hr>

## Contents

* [Features](#features)
* [Getting Started](#getting-started)
    * [Installation](#installation)
    * [Basic Use](#basic-use)
* [Config Composition](#config-composition)
* [Environment Variables](#environment-variables)
* [Using Templates](#using-templates)
* [Usage in Node Modules](#usage-in-node-modules)
* [Customization](#customization)


## Features

* **Simple** - Confippet's `presetConfig` automatically composes a single config
  object from multiple files. For most applications, no further customization is
  necessary.
* **Flexible** - Supports JSON, YAML, and JavaScript config files.
* **Powerful** - Handles complex multi-instance enterprise deployments.


## Getting Started

In this example, we'll create two config files: a default file that always
loads, and a production file that loads only when the `NODE_ENV` environment
variable is set to `production`. We'll then import those files into a standard
Node.js app.

### Installation

```
npm install electrode-confippet --save
```

### Basic Use

Make a `config/` directory inside the main app directory, and put the following
into a file named `default.json` in that directory:

```json
{
  "settings": {
    "db": {
      "host": "localhost",
      "port": 5432,
      "database": "clients"
    }
  }
}
```

Next, add another file called `production.json` to the `config/` directory, with
this content:

```json
{
  "settings": {
    "db": {
      "host": "prod-db-server"
    }
  }
}
```

Finally, in our Node.js app, we can import Confippet and use the configuration
we've created:

```javascript
const config = require("electrode-confippet").config;
const db = config.$("settings.db");
```

In this example, `default.json` will be loaded in all environments, whereas
`production.json` will be loaded only when the `NODE_ENV` environment variable
is set to `production`. In that case, the value of `host` in the `db` object
will be overwritten by the value in `production.json`.


## Config Composition

Confippet's `presetConfig` composes together files in the `config/` directory,
in the following order:

> This is the same as [node-config files].

1. `default.EXT`
1. `default-{instance}.EXT`
1. `{deployment}.EXT`
1. `{deployment}-{instance}.EXT`
1. `{short_hostname}.EXT`
1. `{short_hostname}-{instance}.EXT`
1. `{short_hostname}-{deployment}.EXT`
1. `{short_hostname}-{deployment}-{instance}.EXT`
1. `{full_hostname}.EXT`
1. `{full_hostname}-{instance}.EXT`
1. `{full_hostname}-{deployment}.EXT`
1. `{full_hostname}-{deployment}-{instance}.EXT`
1. `local.EXT`
1. `local-{instance}.EXT`
1. `local-{deployment}.EXT`
1. `local-{deployment}-{instance}.EXT`

Where:

* `EXT` can be any of `["json", "yaml", "js"]`. Confippet will load all of them,
  in that order. Each time it finds a config file, the values in that file will
  be loaded and merged into the config store. So `js` overrides `yaml`, which
  overrides `json`. You can add handlers for other file types and change their
  loading order—see [composeConfig] for further details.
* `{instance}` is your app's instance string in multi-instance deployments
  (specified by the `NODE_APP_INSTANCE` environment variable).
* `{short_hostname}` is your server name up to the first dot.
* `{full_hostname}` is your whole server name.
* `{deployment}` is your deployment environment (specified by the `NODE_ENV`
  environment variable).

Overridden values are handled as follows:

* Objects are merged.
* Primitive values (string, boolean, number) are replaced.
* **Arrays are replaced**, unless the key starts with `+` *and* both the source
  and the target are arrays. In that case, the two arrays are joined together
  using Lodash's `_.union` method.

After Confippet loads all available configuration files, it will look for
override JSON strings from the `NODE_CONFIG` and `CONFIPPET*` environment
variables. See the next section for details.


## Environment Variables

Confippet reads the following environment variables when composing a config
store:

* `AUTO_LOAD_CONFIG_OFF` - If this is set, then Confippet will **not**
  automatically load any configuration into the preset `config` store.
  `Confippet.config` will be an empty store. This enables you to customize the
  config structure before loading.
* `NODE_CONFIG_DIR` - Set the directory to search for config files. By default,
  Confippet looks in the `config` directory for config files.
* `NODE_ENV` - By default, Confippet loads `development` config files after
  loading `default`. Set this environment variable to change to a different
  deployment, such as `production`.
* `NODE_APP_INSTANCE` - If your app is deployed to multiple instances, you can
  set this to load instance-specific configurations.
* `AUTO_LOAD_CONFIG_PROCESS_OFF` - By default, after composing the config from
  all sources, Confippet will use [processConfig] to process
  [templates](#using-templates). You can set this environment variable to
  disable template processing.
* `NODE_CONFIG` - You can set this to a valid JSON string and Confippet will
  parse it to override the configuration.
* `CONFIPPET*` - Any environment variables that starts with `CONFIPPET` will be
  parsed as JSON strings to override the configuration.


## Using Templates

Values in your config files can be templates, which will be resolved with
a preset context. See [processConfig] for more information about how to use
config value templates.


## Usage in Node Modules

If you have a Node.js module that has its own configurations based on
environment variables, like `NODE_ENV`, you can use Confippet to load config
files for your module.

The example below will use the [default compose options] to compose
configurations from the directory `config` under the script's directory
(`__dirname`).

```javascript
const Confippet = require("electrode-confippet");

const options = {
  dirs: [Path.join(__dirname, "config")],
  warnMissing: false,
  context: {
    deployment: process.env.NODE_ENV
  }
};

const defaults = Confippet.store();
defaults._$.compose(options);
```


## Customization

The [composeConfig] feature supports a fully customizable and extendable config
structure. Even Confippet's own preset config structure can be extended, since
it's composed using the same feature.

If you want to use the preset config, but add an extension handler or insert
a source, you can turn off auto loading, and load it yourself with your own
options.

> **NOTE:** This has to happen before any other file accesses
> `Confippet.config`. You should do this in your startup `index.js` file.

For example:

```javascript
process.env.AUTO_LOAD_CONFIG_OFF = true;

const JSON5 = require("json5");
const fs = require("fs");
const Confippet = require("electrode-confippet");
const config = Confippet.config;

const extHandlers = Confippet.extHandlers;
extHandlers.json5 = (fullF) => JSON5.parse(fs.readFileSync(fullF, "utf8"));

Confippet.presetConfig.load(config, {
  extSearch: ["json", "json5", "yaml", "js"],
  extHandlers,
  providers: {
    customConfig: {
      name: "{{env.CUSTOM_CONFIG_SOURCE}}",
      order: 300,
      type: Confippet.providerTypes.required
    }
  }
});
```

The above compose option adds a new provider that looks for a file named by the
environment variable `CUSTOM_CONFIG_SOURCE` and will be loaded after all default
sources are loaded (controlled by `order`).

It also adds a new extension handler, `json5`, to be loaded after `json`.

To further understand the `_$` and the `compose` options, please see the
documentation for [store], [composeConfig], and [processConfig].

Built with :heart: by [Team Electrode](https://github.com/orgs/electrode-io/people) @WalmartLabs.

[node-config npm module]: https://github.com/lorenwest/node-config
[node-config files]: https://github.com/lorenwest/node-config/wiki/Configuration-Files
[store]: ./store.md
[composeConfig]: ./compose.md
[processConfig]: ./templates.md
[default compose options]: ./lib/default-compose-opts.js
[npm-image]: https://badge.fury.io/js/electrode-confippet.svg
[npm-url]: https://npmjs.org/package/electrode-confippet
[travis-image]: https://travis-ci.org/electrode-io/electrode-confippet.svg?branch=master
[travis-url]: https://travis-ci.org/electrode-io/electrode-confippet
[daviddm-image]: https://david-dm.org/electrode-io/electrode-confippet.svg?theme=shields.io
[daviddm-url]: https://david-dm.org/electrode-io/electrode-confippet
