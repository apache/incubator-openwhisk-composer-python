# Compose Command

The `pycompose` command makes it possible to deploy compositions from the command line.

The `pycompose` command is intended as a minimal complement to the OpenWhisk CLI. The OpenWhisk CLI already has the capability to configure, invoke, and delete compositions (since these are just OpenWhisk actions) but lacks the capability to create composition actions. The `compose` command bridges this gap. It makes it possible to deploy compositions as part of the development cycle or in shell scripts. It is not a replacement for the OpenWhisk CLI however as it does not duplicate existing OpenWhisk CLI capabilities. Moreover, for a much richer developer experience, we recommend using [Shell](https://github.com/ibm-functions/shell).

## Usage

```
pycompose
```
```
Usage:
  compose composition.[py|json] command [flags]
Commands:
  --json                 output the json representation for the composition (default command)
  --deploy NAME          deploy the composition with name NAME
  --encode               output the conductor action code for the composition
Flags:
  --lower [VERSION]      lower to primitive combinators or specific composer version
  --apihost HOST         API HOST
  -u, --auth KEY         authorization KEY
  -i, --insecure         bypass certificate checking
```
The `pycompose` command requires either a Python file that evaluates to a composition (for example [demo.py](../samples/demo.py)) or a JSON file that encodes a composition (for example [demo.json](../samples/demo.json)). The JSON format is documented in [FORMAT.md](FORMAT.md).

The `pycompose` command has three modes of operation:
- By default or when the `--json` option is specified, the command returns the composition encoded as a JSON dictionary.
- When the `--deploy` option is specified, the command deploys the composition given the desired name for the composition.
- When the `--encode` option is specified, the command returns the Python code for the [conductor action](https://github.com/apache/incubator-openwhisk/blob/master/docs/conductors.md) for the composition.

## JSON format

By default, the `compose` command evaluates the composition code and outputs the resulting JSON dictionary:
```
pycompose demo.py
```
```json
{
    "type": "if",
    "test": {
        "type": "action",
        "name": "/_/authenticate",
        "action": {
            "exec": {
                "kind": "nodejs:default",
                "code": "const main = function ({ password }) { return { value: password === 'abc123' } }"
            }
        }
    },
    "consequent": {
        "type": "action",
        "name": "/_/success",
        "action": {
            "exec": {
                "kind": "nodejs:default",
                "code": "const main = function () { return { message: 'success' } }"
            }
        }
    },
    "alternate": {
        "type": "action",
        "name": "/_/failure",
        "action": {
            "exec": {
                "kind": "nodejs:default",
                "code": "const main = function () { return { message: 'failure' } }"
            }
        }
    }
}
```
The evaluation context includes the `composer` object implicitly. In other words, there is no need to import the `composer` module explicitly in the composition code.

## Deployment

The `--deploy` option makes it possible to deploy a composition (Javascript or JSON) given the desired name for the composition:
```
compose demo.js --deploy demo
```
```
ok: created actions /_/authenticate,/_/success,/_/failure,/_/demo
```
Or:
```
compose demo.py > demo.json
compose demo.json --deploy demo
```
```
ok: created actions /_/authenticate,/_/success,/_/failure,/_/demo
```
The `pycompose` command synthesizes and deploys a conductor action that implements the
composition with the given name. It also deploys the composed actions for which
definitions are provided as part of the composition.

The `pycompose` command outputs the list of deployed actions or an error result. If an error occurs during deployment, the state of the various actions is unknown.

The `pycompose` command deletes the deployed actions before recreating them if necessary. As a result, default parameters, limits, and annotations on preexisting actions are lost.

### Configuration

Like the OpenWhisk CLI, the `compose` command supports the following flags for specifying the OpenWhisk deployment to use:
```
 --apihost HOST         API HOST
  -u, --auth KEY        authorization KEY
  -i, --insecure        bypass certificate checking
```
If the `--apihost` flag is absent, the environment variable `__OW_API_HOST` is used in its place. If neither is available, the `compose` command extracts the `APIHOST` key from the whisk property file for the current user.

If the `--auth` flag is absent, the environment variable `__OW_API_KEY` is used in its place. If neither is available, the `compose` command extracts the `AUTH` key from the whisk property file for the current user.

The default path for the whisk property file is `$HOME/.wskprops`. It can be altered by setting the `WSK_CONFIG_FILE` environment variable.

## Code generation

The `pycompose` command returns the code of the conductor action for the composition (Python or JSON) when invoked with the `--encode` option.
For instance, the conductor action code for the [demo.py](../samples/demo.py) composition is [demo-conductor.py](../samples/demo-conductor.py):
```
pycompose demo.py --encode > demo-conductor.py
```
This code may be deployed using the OpenWhisk CLI:
```
wsk action create demo demo-conductor.py --kind python:3 -a conductor true
```
```
ok: created action demo
```
The conductor action code does not include definitions for nested actions or compositions.

## Lowering (**Not supported yet**)

If the `--lower VERSION` option is specified, the `pycompose` command uses the set of combinators of the specified revision of the `composer` module. More recently introduced combinators (if any) are translated into combinators of the older set.

If the `--lower` option is specified without a version number, the `pycompose` command uses only primitive combinators.

These options may be combined with any of the `pycompose` commands.