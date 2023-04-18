# Security Considerations

At a high level, `cdc-sink` accepts network connections to apply arbitrary mutations to the target
cluster. In order to limit the scope of access, [JSON Web Tokens](https://jwt.io) are used to
authorize incoming connections and limit them to writing to a subset of the SQL databases or
user-defined schemas in the target cluster.

A minimally-acceptable JWT claim is shown below. It includes the standard `jti` token identifier
field, and a custom claim for cdc-sink. This custom claim is defined in a globally-unique namespace
and most JWT-enabled authorization providers support adding custom claims. It contains a list of one
or more target databases or user-defined schemas. Wildcards for databases or schema names are
supported.

Acceptable JWT tokens must be signed with RSA or EC keys. HMAC and  `None`
signatures are rejected. The PEM-formatted public signing keys must be added to the
`_cdc_sink.jwt_public_keys` table. If a specific token needs to be revoked, its `jti`
value can be added to the `_cdc_sink.jwt_revoked_ids` table. These tables will be re-read every
minute by the `cdc-sink` process. A `HUP` signal can be sent to force an early refresh.

The encoded token is provided to the `CREATE CHANGEFEED` using
the `WITH webhook_auth_header='Bearer <encoded token>'`
[option](https://www.cockroachlabs.com/docs/stable/create-changefeed.html#options). In order to
support older versions of CockroachDB, the encoded token may also be specified as a query
parameter `?access_token=<encoded token>`. Note that query parameters may be logged by intermediate
loadbalancers, so caution should be taken.

## Token Quickstart

This example uses `OpenSSL`, but the ultimate source of the key materials doesn't matter, as long as
you have PEM-encoded RSA or EC keys.

```bash
# Generate a EC private key using OpenSSL.
openssl ecparam -out ec.key -genkey -name prime256v1

# Write the public key components to a separate file.
openssl ec -in ec.key -pubout -out ec.pub

# Upload the public key for all instances of cdc-sink to find it.
cockroach sql -e "INSERT INTO _cdc_sink.jwt_public_keys (public_key) VALUES ('$(cat ec.pub)')"

# Reload configuration, or wait one minute.
killall -HUP cdc-sink

# Generate a token which can write to the ycsb.public schema.
# The key can be decoded using the debugger at https://jwt.io.
# Add the contents of out.jwt to the CREATE CHANGEFEED command:
# WITH webhook_auth_header='Bearer <out.jws>'
cdc-sink make-jwt -k ec.key -a ycsb.public -o out.jwt
```

## External JWT Providers

The `make-jwt` subcommand also supports a `--claim` option, which will simply print a JWT claim
template, which can be signed by your existing JWT provider. The PEM-formatted public key or keys
for that provider will need to be inserted into tho `_cdc_sink.jwt_public_keys` table, as above.
The `iss` (issuers) and `jti` (token id) fields will likely be specific to your auth provider, but
the custom claim must be retained in its entirety.

`cdc-sink make-jwt -a 'database.schema' --claim`:

```json
{
  "iss": "cdc-sink",
  "jti": "d5ffa211-8d54-424b-819a-bc19af9202a5",
  "https://github.com/cockroachdb/cdc-sink": {
    "schemas": [
      [
        "database",
        "schema"
      ]
    ]
  }
}
```