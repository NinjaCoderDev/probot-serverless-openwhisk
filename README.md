# Serverless Probot on Openwhisk
> A wrapper to run a GitHub App built with [Probot](https://probot.github.io) as OpenWhisk action. 

Running a Probot app in OpenWhisk might be a bit challenging as the action invocation params need to
be translated into a probot event. This package offers an easy wrapper to turn an existing
probot app into an OpenWhisk action.

## Status
[![GitHub license](https://img.shields.io/github/license/tripodsan/probot-serverless-openwhisk.svg)](https://github.com/tripodsan/probot-serverless-openwhisk/blob/master/LICENSE.txt)
[![GitHub issues](https://img.shields.io/github/issues/tripodsan/probot-serverless-openwhisk.svg)](https://github.com/tripodsan/probot-serverless-openwhisk/issues)
[![CircleCI](https://img.shields.io/circleci/project/github/tripodsan/probot-serverless-openwhisk.svg)](https://circleci.com/gh/tripodsan/probot-serverless-openwhisk)
[![codecov](https://img.shields.io/codecov/c/github/tripodsan/probot-serverless-openwhisk.svg)](https://codecov.io/gh/tripodsan/probot-serverless-openwhisk)
[![Greenkeeper badge](https://badges.greenkeeper.io/tripodsan/probot-serverless-openwhisk.svg)](https://greenkeeper.io/)
[![LGTM Code Quality Grade: JavaScript](https://img.shields.io/lgtm/grade/javascript/g/tripodsan/probot-serverless-openwhisk.svg?logo=lgtm&logoWidth=18)](https://lgtm.com/projects/g/tripodsan/probot-serverless-openwhisk)

## Setup

1. Create a [Probot](https://probot.github.io) app following github's instructions

2. Add this wrapper as dev dependency:
    ```sh
    # Add OpenWhisk wrapper as dependency 
    npm add probot-serverless-openwhisk
    ```

3. Add an `index.js`:
    ```js
    const { OpenWhiskWrapper } = require('probot-servleless-openwhisk');
    const app = require('./src/probot_app.js');
    const view = require('./src/views/probot.js');
    
    module.exports.main = new OpenWhiskWrapper()
      .withHandler(app)
      .withRoute('/probot', view)
      .create();
    ```

4. Build the OpenWhisk action
    ```sh
    $ ./node_modules/.bin/wskbot
    ...
    Created action: dist/probot-openwhisk-example.zip.
    ```
5. Deploy the OpenWhisk action
    ```sh
    $ wsk action update probot-openwhisk-example --docker tripodsan/probot-ow-nodejs8:latest --web raw dist/probot-openwhisk-example.zip
    ```

6. Set the correct [github app settings](https://github.com/settings/apps):    
    
    * **Homepage URL:** https://adobeioruntime.net/api/v1/web/$WSK_NAMESPACE/default/probot-openwhisk-example/probot
    * **Webhook URL:** https://adobeioruntime.net/api/v1/web/$WSK_NAMESPACE/default/probot-openwhisk-example

## Passing the OpenWhisk action params into the handler

Sometimes the handler needs access to the action params, especially if they are deployment provided
and might contain keys or tokens. This wrapper passes them along when it initializes the handler with an additional
argument. 

Example:

```js
module.exports = (app, actionParams = {}) => {
  app.log('Yay, my app is loaded');

  const MY_TOKEN = actionParams.MY_TOKEN || '';
  .
  .
```

The deploy parameters can be specifies in the CLI via `-p`. See below.

## CLI

The command line interface `wskbot` can either be invoked via `./node_modules/.bin/wskbot`. 
you can also use npx: `npx wskbot` or install it globally `npm install -g probot-serverless-openwhisk`.

```
$ wskbot --help
Options:
  --version            Show version number                             [boolean]
  --verbose, -v                                                 [default: false]
  --deploy             Automatically deploy to OpenWhisk        [default: false]
  --test               Invoke action after deployment           [default: false]
  --hints, --no-hints  Show action and github app settings       [default: true]
  --static, -s         Includes a static file into the archive
                                                           [array] [default: []]
  --params, -p         Include the given action param. Can be a file or a
                       string; can be json or env.         [array] [default: []]
  --help               Show help                                       [boolean]

for more information, find our manual at
https://github.com/tripodsan/probot-serverless-openwhisk
```

With no arguments,the `wskbot` just bundles your code into the respective `action.zip`:

```
$ wskbot
ok: created action: dist/probot-openwhisk-example.zip.
Deploy to openwhisk the following command or specify --deploy on the commandline:
$ wsk action update probot-openwhisk-example --docker tripodsan/probot-ow-nodejs8:latest --web raw dist/probot-openwhisk-example.zip

Githup App Settings:
Homepage URL: https://adobeioruntime.net/api/v1/web/tripod/default/probot-openwhisk-example/probot
 Webhook URL: https://adobeioruntime.net/api/v1/web/tripod/default/probot-openwhisk-example
```

### Automatically deploy to openwhisk

When given the `--deploy`, the `wskbot` will try to deploy it ot OpenWhisk using the settings from
`~/.wskprops`. Alternatively, you can also set the `WSK_NAMESPACE`, `WSK_AUTH`, `WSK_APIHOST` in your
environment or `.env` file.

```
$ wskbot --deploy --no-hints
ok: created action: dist/probot-openwhisk-example.zip.
ok: updated action tripod/probot-openwhisk-example
```  

### Automatically _test_ the deployed action

In order to quickly test the deployed action, `wskbot` can send a `GET` request to the action url.

```
$ wskbot --deploy --no-hints --test
ok: created action: dist/probot-openwhisk-example.zip.
ok: updated action tripod/probot-openwhisk-example
--: requesting: https://runtime.adobe.io/api/v1/web/tripod/default/probot-openwhisk-example ...
ok: 200
```

..or sometimes:

```
$ wskbot --deploy --no-hints --test
ok: created action: dist/probot-openwhisk-example.zip.
ok: updated action tripod/probot-openwhisk-example
--: requesting: https://runtime.adobe.io/api/v1/web/tripod/default/probot-openwhisk-example ...
error:  400 - "{\n  \"error\": \"Response is not valid 'message/http'.\",\n  \"code\": \"av6qzDTHdgd5dfg7WOynEjbVnTdE5JhnB4c\"\n}"
```

### Including action parameters

Action parameters can be defined via `-p`, either as json on env string, or json or env file.

Examples:

```bash
# specify as env string
wskbot -p MY_TOKEN=1234 -p MY_PWD=foo

# specify as json string
wskbot -p '{ "MY_TOKEN": 1234, "MY_PWD": "foo" }'

# specify as env file
wskbot -p .env

# specify as json file
wskbot -p params.json

# and a combination of the above
wskbot -p .env -p params.json -p MY_TOKEN=123
```

### Including static files

Adding static files, i.e. files that are not referenced from the `index.js` and detected by webpack,
can be done via the `-s` parameter. they are always put into the root directory of the archive.

Example:

```bash
# include an image
wskbot -s logo.png
```
 
## Notes

### Bundling

The action is created using webpack to create bundle for the sources and then creates a zip archive
with the bundle, a `package.json`, the private key files and the `.env`.

### Docker Image

The action needs a special docker image [tripodsan/probot-ow-nodejs8:latest](https://github.com/tripodsan/probot-openwhisk-docker)
that already contains `probot` and its dependencies. This helps to keep the size of the action small. 

## Contributing

If you have suggestions for how this OpenWhisk probot wrapper could be improved, or want to report a bug, open an issue! We'd love all and any contributions.

For more, check out the [Contributing Guide](CONTRIBUTING.md).

