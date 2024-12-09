# driver-did-indy

DID Registration Driver for did:indy.

This driver is designed to be used standalone or with a universal registrar.

When used as a standalone service, access to the service can be restricted to registered parties.

## Dynamic Client Registration

### OAuth Metadata

The driver publishes metadata about the service using the endpoint defined by [OAuth 2.0 Authorization Server Metadata][asmeta], `/.well-known/oauth-authorization-server`.

Example response:

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "issuer": "https://driver-did-indy.example.com",
  "token_endpoint": "https://driver-did-indy.example.com/token",
  "token_endpoint_auth_methods_supported": ["private_key_jwt"],
  "grant_types_supported": ["client_credentials"],
  "jwks_uri": "https://driver-did-indy.example.com/jwks.json",
  "registration_endpoint": "https://driver-did-indy.example.com/register",
  "scopes_supported": ["all", "nym", "schema", "cred_def", "rev_reg_def", "rev_reg_entry"],
  "service_documentation": "http://driver-did-indy.example.com/docs"
}
```

### Client Registration

Client registration is achieved by the following steps:
1. The Endorser generates a registration token
2. The token is delivered to the client (out-of-band)
3. The Client makes a request to the registration endpoint with the registration token

#### Registration Token Generation

Generation of registration tokens must be done by authorized users only. This can be achieved a number of ways. Three mechanisms are outlined below.

##### Web UI

The Endorser Service could host an Administration Web UI. Users must be authenticated and then authorized to access the token registration function. This could be handled through direct management of users or through an external ID Provider (e.g. Keycloak, Google Enterprise, Microsoft Teams, etc.).

##### CLI

A CLI application with access to the same secrets held by the Endorser Service could be used by an administrator to generate a registration token.

##### After Presentation of a VC from a trusted Issuer

Upon presentation of a credential from a trusted Issuer, the Endorser Service could issue a registration token based on the credential.

#### Usage of a Token vs a VC

Verifiable Credentials are very similar to JWTs in a number of ways. Some VCs are secured using JWTs, even.

Verifiable Credentials enable a transfer of trust from one context to another, in addition to preserving the integrity of a set of claims. This enables a credential issued as proof of an individuals privilege to operate a vehicle to be reused to prove that they are above the threshold required to purchase age restricted goods.

JWTs are a more primitive construct. It is up to the application to define their significance. In this case, the intended use of the registration token is limited to accessing the registration endpoint. It would be inappropriate to reuse the token in any other context. It would just add unnecessary complexity to the token to incorporate requirements of one of the many VC Formats. Therefore, the registration token is deliberately just a simple JWT and not a VC.

#### Registration Token

The Registration Token is a JWT secured by either HS256 (hmac) or an asymmetric signature. When using HS256, the Endorser service itself must be the issuer of the token. When an asymmetric signature is used, the token must be signed by a service trusted by the Endorser (e.g. through service configuration).

The token payload must contain the following claims:

- `iss`: The URL of the Issuer (Endorser or another trusted service)
- `aud`: MUST be the URL of the Endorser
- `iat`: Time of issuance
- `exp`: Expiration time. Service should default to a reasonable time frame. 1 hour is suggested.
- `ver`: An integer representing the version of this registration token. `1` for this version.
- `auto_endorse`: An object with the following keys:
    - `nym_new`: An integer value indicating the number of new nyms the driver will automatically endorse from the client. Defaults to `1` if unset or `auto_endorse` is unset.
    - `nym_update`: A boolean value indicating whether the driver will automatically endorse nym update transactions, excluding role changes, from the client. Defaults to `true` if unset or `auto_endorse` is unset.
    - `nym_role_change`: A boolean value indicating whether the driver will automatically endorse nym update transactions updating roles from the client. This essentially covers nym transactions updating verkey and/or diddocContent. Defaults to `false` if unset or `auto_endorse` is unset.
    - `schema`: A boolean value indicating whether the driver will automatically endorse schema transactions from the client. Defaults to `false` if unset or `auto_endorse` is unset.
    - `cred_def`: A boolean value indicating whether the driver will automatically endorse cred_def transactions from the client. Defaults to `true` if unset or `auto_endorse` is unset.
    - `rev_reg_def`: A boolean value indicating whether the driver will automatically endorse rev_reg_def transactions from the client. Defaults to `true` if unset or `auto_endorse` is unset.
    - `rev_reg_entry`: A boolean value indicating whether the driver will automatically endorse rev_reg_entry transactions from the client. Defaults to `true` if unset or `auto_endorse` is unset.
- `permitted_roles`: A list of roles permitted for nyms submitted by the client. Defaults to an empty list if unset, indicating the client is only permitted to create nyms with the least privileged role on the network, usually called the "client" role.
- `txn_webhook_url`: The URL at which the client will receive webhooks regarding transaction requests.

Example payload:

```json
{
  "iss": "https://driver-did-indy.example.com",
  "aud": "https://driver-did-indy.example.com",
  "iat": 1728060682,
  "exp": 1728064282,
  "ver": 1,
  "auto_endorse": {
    "nym_new": 1,
    "nym_update": true,
    "schema": true,
    "cred_def": true
  },
  "txn_webhook_url": "https://indy-client.example.com"
}
```

##### Binding the registration token to a sender

When a public key is known for the client at the time of token generation, the token MAY be sender-constrained. Sender-constrained token payloads contain the following claim:

- `cnf`: Token confirmation claim as defined in [RFC 7800][popsem]. It MUST contain `jkt` confirmation method defined in [RFC 9449][dpop]. This claim is a JWK Thumbprint of the client's public key that they will use to later authenticate themselves to the Endorser as the intended sender of the token.

The resulting token is considered a DPoP token (rather than a Bearer token).

When the token is not sender-constrained, the token is a Bearer token.

##### Token Delivery

The Token is delivered to the Client out-of-band. The exact mechanism is out of scope for this document.

#### Registration Endpoint

Upon receiving a Registration Token, the Client makes a request to the registration endpoint. The registration endpoint is a [Dynamic Client Registration][dynclient] Endpoint as defined by RFC 7591.

The Endorser expects the following client metadata properties:

- `client_name` (string): The client's name

##### Registration Request

If the registration token is sender-constrained, the request MUST use `DPoP` authorization and include a proof of possession in the `DPoP` header.

Example DPoP Request:

```http
POST /register
Host: driver-did-indy.example.com 
Content-Type: application/json
Authorization: DPoP <registration token>
DPoP: <base64url encoded DPoP token>
```
```json
{
    "client_name": "My Example Client",
}
```

Both the registration token and the DPoP token must be verified.

If the registration token is unconstrained, the request MUST use `Bearer` authorization.

Example Bearer Request:

```http
POST /register
Host: driver-did-indy.example.com 
Content-Type: application/json
Authorization: Bearer <registration token>
```
```json
{
  "client_name": "My Example Client",
  "grant_types": ["client_credentials"],
  "token_endpoint_auth_method": "private_key_jwt",
  "jwks": {
    "keys": [
      {
        "kty": "OKP",
        "crv": "Ed25519",
        "x": "vcpo2lmlhn6iwku8mkvslg2zaoc-nloypvqncpq447g",
        "use": "sig",
        "kid": "ed25519-key-id-123"
      }
    ]
  }
}
```

##### Registration Response

The registration response contains the `client_id` and `client_secret` that will be used by this client to authenticate itself to the driver.

```http
HTTP/1.1 201 Created
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache
```

```json
{
  "client_id": "s6BhdRkqt3",
  "client_name": "My Example Client",
  "jwks": {
    "keys": [
      {
        "kty": "OKP",
        "crv": "Ed25519",
        "x": "vcpo2lmlhn6iwku8mkvslg2zaoc-nloypvqncpq447g",
        "use": "sig",
        "kid": "ed25519-key-id-123"
      }
    ]
  },
  "client_id_issued_at": 2893256800,
  "grant_types": ["client_credentials"],
  "token_endpoint_auth_method": "private_key_jwt",
  "txn_webhook_url": "https://indy-client.example.com"
}
```

### Client Authentication

The client authenticates to the driver using the `client_credentials` grant type defined in [OAuth 2.0][oauth2].

#### Token Request

To authenticate, the client uses a client assertion type of `urn:ietf:params:oauth:client-assertion-type:jwt-bearer`, providing a signed JWT to the token endpoint.

**Constructing the JWT**:

- **Header**:

  ```json
  {
    "alg": "EdDSA",
    "kid": "ed25519-key-id-123"
  }
  ```

- **Payload**:

  ```json
  {
    "iss": "your-assigned-client-id",
    "sub": "your-assigned-client-id",
    "aud": "https://driver-did-indy.example.com",
    "exp": 1698766000,
    "iat": 1698762400
  }
  ```

**Example Token Request**:

```http
POST /token HTTP/1.1
Host: driver-did-indy.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&
client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer&
client_assertion=eyJhb...
```

#### Token Response

The response is a JWT Access Token to be used at the Endorser API Endpoints defined below.

---

## Endpoints

The driver provides the following endpoints for DID registration operations as per the [DID Registration][didreg] specification.

### POST `/create`

#### Create a DID

This endpoint creates a DID. It takes the DID document, plus optional DID registration [options][didreg-options] and [secrets][didreg-secret] needed to create a DID. The output is a state object representing the current state of the DID creation process.

**Endpoint**: `POST /create?method={method}`

- **Method**: `POST`
- **Query Parameters**:
  - `method` (string, required): The requested DID method for the operation. Example: `indy`

**Request Body**:

- `jobId` (string, optional): A unique identifier to track the operation.
- `didDocument` (object, required): The DID document to be registered.
- `options` (object, optional): Registration options.
- `secret` (object, optional): Secrets needed for DID creation.

**Example Request**:

```http
POST /create?method=indy HTTP/1.1
Host: driver-did-indy.example.com
Content-Type: application/json
Authorization: Bearer <access_token>
```

```json
{
  "didDocument": {
    "id": "did:indy:sovrin:WRfXPg8dantKVubE3HX8pw",
    "controller": ["did:indy:sovrin:WRfXPg8dantKVubE3HX8pw"],
    "verificationMethod": [...],
    "service": [...]
  },
  "options": {
    "network": "testnet"
  },
  "secret": {
    "seed": "72WGp7NgFR1Oqdi8zlt7jQQ434XR0cNQ"
  }
}
```

**Responses**:

- **201 Created**: The DID has been successfully created.
- **200 OK**: The request was successful, but the DID may not be fully created yet.
- **400 Bad Request**: Invalid input fields.
- **500 Internal Server Error**: An internal error occurred.

**Example Response**:

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "jobId": "6d85bcd0-2ea3-4288-ab00-15afadd8a156",
  "didState": {
    "state": "finished",
    "did": "did:indy:sovrin:WRfXPg8dantKVubE3HX8pw",
    "secret": {
      "seed": "72WGp7NgFR1Oqdi8zlt7jQQ434XR0cNQ"
    },
    "didDocument": {
      "id": "did:indy:sovrin:WRfXPg8dantKVubE3HX8pw",
      "controller": ["did:indy:sovrin:WRfXPg8dantKVubE3HX8pw"],
      "verificationMethod": [...],
      "service": [...]
    }
  },
  "didRegistrationMetadata": { },
  "didDocumentMetadata": { }
}
```

### POST `/update`

#### Update a DID

This endpoint updates an existing DID. It requires the DID, an operation to perform on the DID document, and optional registration options and secrets.

**Endpoint**: `POST /update`

- **Method**: `POST`

**Request Body**:

- `jobId` (string, optional): A unique identifier to track the operation.
- `did` (string, required): The DID to be updated.
- `didDocumentOperation` (array, required): The operations to perform (e.g., `setDidDocument`).
- `didDocument` (object, required): The DID document or changes to apply.
- `options` (object, optional): Registration options.
- `secret` (object, optional): Secrets needed for DID update.

**Example Request**:

```http
POST /update HTTP/1.1
Host: driver-did-indy.example.com
Content-Type: application/json
Authorization: Bearer <access_token>
```

```json
{
  "did": "did:indy:sovrin:WRfXPg8dantKVubE3HX8pw",
  "didDocumentOperation": ["setDidDocument"],
  "didDocument": {
    "service": [...]
  },
  "options": {
    "network": "testnet"
  },
  "secret": {
    "seed": "72WGp7NgFR1Oqdi8zlt7jQQ434XR0cNQ"
  }
}
```

**Responses**:

- **200 OK**: The request was successful; the DID may or may not be fully updated yet.
- **400 Bad Request**: Invalid input fields.
- **500 Internal Server Error**: An internal error occurred.

**Example Response**:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "jobId": "6d85bcd0-2ea3-4288-ab00-15afadd8a156",
  "didState": {
    "state": "finished",
    "did": "did:indy:sovrin:WRfXPg8dantKVubE3HX8pw",
    "didDocument": {
      "id": "did:indy:sovrin:WRfXPg8dantKVubE3HX8pw",
      "service": [...]
    }
  },
  "didRegistrationMetadata": { },
  "didDocumentMetadata": { }
}
```

### POST `/deactivate`

#### Deactivate a DID

This endpoint deactivates an existing DID.

**Endpoint**: `POST /deactivate`

- **Method**: `POST`

**Request Body**:

- `jobId` (string, optional): A unique identifier to track the operation.
- `did` (string, required): The DID to be deactivated.
- `options` (object, optional): Registration options.
- `secret` (object, optional): Secrets needed for DID deactivation.

**Example Request**:

```http
POST /deactivate HTTP/1.1
Host: driver-did-indy.example.com
Content-Type: application/json
Authorization: Bearer <access_token>
```

```json
{
  "did": "did:indy:sovrin:WRfXPg8dantKVubE3HX8pw",
  "options": {
    "network": "testnet"
  },
  "secret": {
    "seed": "72WGp7NgFR1Oqdi8zlt7jQQ434XR0cNQ"
  }
}
```

**Responses**:

- **200 OK**: The request was successful; the DID may or may not be fully deactivated yet.
- **400 Bad Request**: Invalid input fields.
- **500 Internal Server Error**: An internal error occurred.

**Example Response**:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "jobId": "6d85bcd0-2ea3-4288-ab00-15afadd8a156",
  "didState": {
    "state": "finished",
    "did": "did:indy:sovrin:WRfXPg8dantKVubE3HX8pw"
  },
  "didRegistrationMetadata": { },
  "didDocumentMetadata": { }
}
```

### POST `/execute`

#### Execute an Operation

This endpoint executes a custom operation on a DID that is not covered by standard create/update/deactivate operations.

**Endpoint**: `POST /execute`

- **Method**: `POST`

**Request Body**:

- `jobId` (string, optional): A unique identifier to track the operation.
- `did` (string, required): The DID on which to execute the operation.
- `operation` (array, required): The operation(s) to execute.
- `operationData` (array, optional): Additional data for the operation.
- `options` (object, optional): Registration options.
- `secret` (object, optional): Secrets needed for the operation.

**Example Request**:

```http
POST /execute HTTP/1.1
Host: driver-did-indy.example.com
Content-Type: application/json
Authorization: Bearer <access_token>
```

```json
{
  "did": "did:indy:sovrin:WRfXPg8dantKVubE3HX8pw",
  "operation": ["customOperation"],
  "operationData": [{ "key": "value" }],
  "options": {
    "network": "testnet"
  },
  "secret": {
    "seed": "72WGp7NgFR1Oqdi8zlt7jQQ434XR0cNQ"
  }
}
```

**Responses**:

- **200 OK**: The request was successful; the operation may or may not be fully executed yet.
- **400 Bad Request**: Invalid input fields.
- **500 Internal Server Error**: An internal error occurred.

**Example Response**:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "jobId": "6d85bcd0-2ea3-4288-ab00-15afadd8a156",
  "didState": {
    "state": "finished",
    "did": "did:indy:sovrin:WRfXPg8dantKVubE3HX8pw"
  },
  "operationResult": [{ "resultKey": "resultValue" }],
  "didRegistrationMetadata": { },
  "didDocumentMetadata": { }
}
```

### GET `/properties`

#### Retrieve Configuration Properties

This endpoint returns a map of the configuration properties of the DID Registrar, including its drivers.

**Endpoint**: `GET /properties`

- **Method**: `GET`

**Responses**:

- **200 OK**: Success. Returns a JSON object containing configuration properties.

**Example Response**:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "driver": "did-indy",
  "network": "sovrin",
  "version": "1.0.1"
}
```

### GET `/methods`

#### List Supported DID Methods

This endpoint returns a list of DID methods supported by the DID Registrar.

**Endpoint**: `GET /methods`

- **Method**: `GET`

**Responses**:

- **200 OK**: Success. Returns an array of supported DID methods.

**Example Response**:

```http
HTTP/1.1 200 OK
Content-Type: application/json

[
  "indy",
  "ebsi",
  "ion",
  "cheqd",
  "key"
]
```

---

## References

- [RFC 6749: OAuth 2.0 Authorization Framework][oauth2]
- [RFC 7591: Dynamic Client Registration][dynclient]
- [RFC 9449: OAuth 2.0 Demonstrating Proof of Possession][dpop]
- [RFC 7800: OAuth 2.0 Proof-of-Possession Key Semantics for JSON Web Tokens][popsem]
- [RFC 8414: OAuth 2.0 Authorization Server Metadata][asmeta]
- [Decentralized Identifiers (DIDs) v1.0][didcore]
- [DID Registration Specification][didreg]
- [DID Registration Options][didreg-options]
- [DID Registration Secrets][didreg-secret]

[didcore]: https://www.w3.org/TR/did-core/
[oauth2]: https://datatracker.ietf.org/doc/html/rfc6749
[dpop]: https://datatracker.ietf.org/doc/html/rfc9449
[popsem]: https://datatracker.ietf.org/doc/html/rfc7800
[asmeta]: https://datatracker.ietf.org/doc/html/rfc8414
[dynclient]: https://datatracker.ietf.org/doc/html/rfc7591
[didreg]: https://identity.foundation/did-registration/
[didreg-options]: https://identity.foundation/did-registration/#options
[didreg-secret]: https://identity.foundation/did-registration/#secret
