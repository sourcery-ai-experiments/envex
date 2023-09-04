# ENV EXtended

`envex` is a dotenv `.env` aware environment variable handler with typing features and Hashicorp vault support.

[![PyPI version](https://badge.fury.io/py/envex.svg)](https://badge.fury.io/py/envex)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](https://opensource.org/licenses/MIT)

## Overview

This module provides a convenient interface for handling the environment, and therefore configuration of any application
using 12factor.net principals removing many environment-specific variables and security sensitive information from
application code.

An `Env` instance delivers a lot of functionality by providing a type-smart front-end to `os.environ`,
providing a superset of `os.environ` functionality, including setting default values.

From version 2.0, this module also supports transparently fetching values from Hashicorp vault,
which reduces the need to store secrets in plain text on the filesystem.
This functionality is optional, activated automatically when the `hvac` module is installed, and connection and
authentication to Vault succeed.
Only get (no set) operations to Vault are supported.
Values fetched from Vault are cached by default to reduce the overhead of the api call.
If this is of concern to security, caching can be disabled using the `enable_cache=False` parameter to Env.

This module provides some features not supported by other dotenv handlers (python-dotenv, etc.) including recursive
expansion of template variables, which can be very useful for DRY.

```python
from envex import env

assert env['HOME'] == '/Users/username'

env['TESTING'] = 'This is a test'
assert env.get('TESTING') == 'This is a test'
assert env['TESTING'] == 'This is a test'
assert env('TESTING') == 'This is a test'

import os

assert os.environ['TESTING'] == 'This is a test'

assert env.get('UNSET_VAR') is None
env.set('UNSET_VAR', 'this is now set')
assert env.get('UNSET_VAR') is not None
env.setdefault('UNSET_VAR', 'and this is a default value but only if not set')
assert env.get('UNSET_VAR') == 'this is now set'
del env['UNSET_VAR']
assert env.get('UNSET_VAR') is None
```

Note that there is a subtle difference between `env.get(<variable>, default=<default value>)`
and `env(<variable>, default=<default value>)`.
If the variable is unset, the former simply returns the default value,
but the latter also sets the value to the default value in the environment.

An Env instance can also read a `.env` (default name) file and update the application environment accordingly.
It can read this either when created with `Env(readenv=True)` or directly by using the method `read_env()`.
To override the base name of the dot env file, use the `DOTENV` environment variable.
Other kwargs that can be passed to `Env` when created:

* environ (env): pass the environment to update, default is os.environ, passing an empty dict will create a new env
* readenv (bool): search for and read .env files
* env_file (str): name of the env file, `os.environ["DOTENV"]` if set, or `.env` by default
* search_path (str or list): a single path or list of paths to search for the env file
  search_path may also be passed as a colon-separated list (or semicolon on Windows) of directories to search.
* parents (bool): search (or not) parents of dirs in the search_path
* overwrite (bool): overwrite already set values read from .env, default is to only set if not currently set
* update (bool): push updates os.environ if true (default) otherwise pool changes internally only
* working_dirs (bool): add CWD for the current process and PWD of source .env file
* exception: (optional) Exception class to raise on error (default is `KeyError`)
* errors: bool whether to raise error on missing env_file (default is False)
* kwargs: (keyword args, optional) additional environment variables to add/override

In addition, Env supports a few HashiCorp Vault configuration parameters:

* url: (str, optional) vault url, default is `$VAULT_ADDR`
* token: (str, optional) vault token, default is `$VAULT_TOKEN` or content of `~/.vault-token`
* cert: (tuple, optional) (cert, key) path to client certificate and key files
* verify: (bool, optional) whether to verify server cert (default is True)
* cache_enabled: (bool, optional) whether to cache secrets (default is True)
* base_path: (optional) str base path, or "environment" for secrets (default is None).
  This is used to prefix the path to the secret, i.e. `f"/secret/{base_path}/key"`.

Some type-smart functions act as an alternative to `Env.get` and having to parse the result:

```python
from envex import env

env['AN_INTEGER_VALUE'] = 2875083
assert env.get('AN_INTEGER_VALUE') == '2875083'
assert env.int('AN_INTEGER_VALUE') == 2875083
assert env('AN_INTEGER_VALUE', type=int) == 2875083

env['A_TRUE_VALUE'] = True
assert env.get('A_TRUE_VALUE') == 'True'
assert env.bool('A_TRUE_VALUE') is True
assert env('A_TRUE_VALUE', type=bool) is True

env['A_FALSE_VALUE'] = 0
assert env.get('A_FALSE_VALUE') == '0'
assert env.int('A_FALSE_VALUE') == 0
assert env.bool('A_FALSE_VALUE') is False
assert env('A_FALSE_VALUE', type=bool) is False

env['A_FLOAT_VALUE'] = 287.5083
assert env.get('A_FLOAT_VALUE') == '287.5083'
assert env.float('A_FLOAT_VALUE') == 287.5083
assert env('A_FLOAT_VALUE', type=float) == 287.5083

env['A_LIST_VALUE'] = '1,"two",3,"four"'
assert env.get('A_LIST_VALUE') == '1,"two",3,"four"'
assert env.list('A_LIST_VALUE') == ['1', 'two', '3', 'four']
assert env('A_LIST_VALUE', type=list) == ['1', 'two', '3', 'four']
```

Environment variables are always stored as strings.
This is enforced by the underlying os.environ, but also true of any
provided environment, which must use the `MutableMapping[str, str]` contract.

## Vault

Env supports fetching secrets from Hashicorp Vault using the `kv.v2` engine.
This is a convenient way to store secrets securely, avoiding the need to be in plain text on the filesystem,
especially plain text files that will be embedded in published docker images.
It is also a good idea to avoid storing secrets in the operating system's environment, which is open to inspection b
external processes, although `Env(readenv=True)` already side-steps that.

Setting up and configuring a vault server is beyond the scope of this document,
but well worth the investment wherever security of concern (i.e. almost always).

See:

- [hashicorp.com](https://developer.hashicorp.com/vault) for the developer documentation and detailed information and
  tutorials on setting up your own vault server, and
- [vaultproject.io](https://www.vaultproject.io/) for information about HashiCorp's managed cloud offering.

The role assigned to the token used to access the vault server needs it only have `read` capability for the path to the
secret, and this restriction, and constraint to reading the root path of those secrets, is the recommended policy for
tokens used at runtime by the application.

The utility `env2hvac` is provided to import (create or update) a typical `.env` file into vault based on a prefix
comprised of an application and environment name, using the format `<appname>/<envname>/<key>`.
This utility requires that the `create` role is available to the token used to access the vault server.
Assuming that the `kv` secrets engine is mounted at `secret/`,
the final path at which secrets are stored will be `secret/data/<appname>/<envname>/<key>`.
