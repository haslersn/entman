# entman

LDAP-authorized access to an HTTP endpoint.
(Currently only `POST` requests are supported and the response is dropped.)

## Build

```bash
$ cargo build
```

### With Nix

```bash
$ nix-build
```

With Nix you can also easily cross-compile entman:

```bash
$ nix-build '<nixpkgs>' \
    --arg crossSystem '{ config = "aarch64-unknown-linux-gnu"; }' \
    --arg overlays '[ (self: super: { entman = super.callPackage ./. {}; }) ]' \
    -A entman
```

## Configuration

In the working directory where entman is executed, there must be a
`entman.toml` configuration file.
A good start is to copy the `entman.toml.example` from this repository.

### [server] section

#### `mount_point =`

Prefix to the paths where the [HTTP endpoints](#http-endpoints) are mounted.

#### `port =`

The port on which the [HTTP endpoints](#http-endpoints) are served.

### [client] section

#### `endpoint =`

The URL to the endpoint accessed in case of a successful authorization.
Example: `"http://localhost:8020/castle/lock?toggle"`

### [json_history] section

#### `filename =`

Path to the log file to write to.
This can be an absolute or relative path.
A relative path is relative to the working directory where entman is executed.
See also [#log](#log).

### [ldap] section

#### `url =`

Example: `"ldap://localhost:389"`

#### `base_dn =`

Example: `"dc=example"`

#### `bind_dn =`

Example: `"cn=admin,dc=example"`

#### `bind_password =`

Example: `"12345"`

#### `user_filter =`

The filter used when querying the LDAP server.
Hereby `%t` is replaced by the access token.
Example: `"(accessToken=%t)"`

#### `user_name_attr =`

In the LDAP response, the name of the attribute that contains the username
assigned to the access token.
This is only used so as to log the username to the [log](#log).
Example: `"uid"`

## HTTP endpoints

entman provides an HTTP endpoint `/access`.
For example, with `mount_point = "/entman"` and `port = 8010`, it is accessible
at `http://localhost:8010/entman/access`.
We use these configuration values in the further examples.

We have a subsection for each type of supported request:

### `GET` request to the `/access` endpoint

Such a request returns the [log](#log) or parts of it.

```bash
$ curl -X GET http://localhost:8010/entman/access?time_min=1546297200&time_max=1577833199&token=foo&name=jane-doe&outcome=success&only_latest=false
{"time":1546297200,"token":"foo","response":{"outcome":"Success","name":"jane-doe"}}
```

Hereby each of the query parameters is optional.
They have the following meanings.

* `time_min=`: Mimimum `"time"` of returned lines.
* `time_max=`: Maximum `"time"` of returned lines.
* `token=`: Filter by `"token"`.
* `outcome=`: Filter by lower case version of the `"outcome"`.
* `name=`: Filter by `"name"`.

### `POST` request to the `/access` endpoint

This is the main feature of this program.

```bash
$ curl -X POST http://localhost:8010/entman/access?token=foo
```

Such a request performs an access using a provided token.
An LDAP query is performed using the configured `user_filter`
where `%t` is replaced by the token.
If that query produces exactly one match, then the access is considered to be
successful and entman performs a `POST` request to the configured
`[client] endpoint`.
The response to that `POST` request is dropped.

## Log

For each access a line is appended to the log file.
Such a line looks as follows.

```json
{"time":1546297200,"token":"foo","response":{"outcome":"Success","name":"jane-doe"}}
```

Hereby:

* `"time"` is the timestamp of the access.
* `"token"` is the token used
* `"response"`
  * `"outcome"` is one of `"Unknown"`, `"Success"` and `"Revoked"`.
      * `"Unknown"` means that the token was not matched.
        In that case `"name"` is `null`.
      * `"Success"` means that the token was matched exactly once.
        An attempt to access the `[client] endpoint` has subsequently been made.
        However, this does not tell anything about whether that attempt itself was
        successful.
      * `"Revoked"` means that the token was matched more than once.
        In that case `"name"` is `null`.
  * `"name"` is, in case of a successful outcome, the name of the user whose token
  was matched.
