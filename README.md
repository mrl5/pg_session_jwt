pg\_session\_jwt
================

`pg_session_jwt` is a PostgreSQL extension designed to handle authenticated sessions through a JWT. When configured with a JWK (JSON Web Key), it verifies JWT authenticity. When operating without a JWK, it can fall back to using PostgREST-compatible JWT claims parameter.

**JWK can only be set at postmaster startup, from the configuration file, or by client request in the connection startup packet** (e.g., from libpq's PGOPTIONS variable), whereas the JWT can be set anytime at runtime. When a JWK is configured, the extension stores the JWT in the database for later retrieval and exposes functions to retrieve the user ID (the `sub` subject field) and other parts of the payload. When no JWK is configured, it falls back to using the PostgREST-compatible `request.jwt.claims` parameter for user identification.

The goal of this extension is to provide a secure and flexible way to manage authenticated sessions in a PostgreSQL database. The JWTs can be generated by third-party auth providers, and then developers can leverage either the validated JWT (when using JWK) or PostgREST-compatible JWT claims (when not using JWK) for [Row Level Security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html) (RLS) policies, or to retrieve the user ID for other purposes (column defaults, filters, etc.).

This extension powers [Neon RLS](https://neon.tech/docs/guides/neon-rls), but it is portable to any Postgres provider or self-managed Postgres deployments.

> [!WARNING]
> This extension is under active development. The API is subject to change.

Features
--------

* **Initialize JWT sessions** using a JWK (JSON Web Key) for secure JWT validation.

* **Flexible authentication modes** - use either JWK-validated JWTs or PostgREST-compatible JWT claims.

* **Retrieve the user ID** or other session-related information directly from the database.

* Simple JSONB-based storage and retrieval of session information.

Usage
-----

The extension can be used in two modes: with JWK validation or with PostgREST-compatible JWT claims.

### Using with JWK Validation

When using JWK validation, you need to initialize the `pg_session_jwt.jwk` parameter before using the extension. This can be done using [libpq connect options](https://www.postgresql.org/docs/current/libpq-connect.html#LIBPQ-CONNECT-OPTIONS):

```console
MY_JWK=...
export PGOPTIONS="-c pg_session_jwt.jwk=$MY_JWK"
```

In this mode, you'll need to:
1. Initialize the session with `auth.init()`
2. Set the JWT using `auth.jwt_session_init(jwt)`
3. Use `auth.user_id()` or `auth.session()` to access the validated JWT data

### Using with PostgREST-compatible JWT Claims

When operating without JWK, the extension works out of the box with PostgREST-compatible JWT claims. No initialization is needed - simply ensure your JWT claims are available as `request.jwt.claims` parameter and use `auth.user_id()` to access the subject claim.

This mode provides seamless integration with PostgREST's JWT handling, allowing you to use the same JWT claims for both PostgREST and database functions.

> [!CAUTION]  
> Security Consideration: When using the fallback mode (without JWK), be aware that `request.jwt.claims` is a regular PostgreSQL parameter that can be modified by any database user. This means users could potentially impersonate others by changing this value.
>
> PostgREST handles this securely by setting these claims in a protected context before executing user queries. If you're not using PostgREST, you must ensure these claims are set in a secure way that prevents unauthorized modifications.

Functions
--------

`pg_session_jwt` exposes four main functions:

### 1\. auth.init() → void

Initializes a session using JWK stored in `pg_session_jwt.jwk` [run-time parameter](https://www.postgresql.org/docs/current/sql-show.html). Only needed when using JWK validation mode.

### 2\. auth.jwt\_session\_init(jwt text) → void

Initializes the JWT session with the provided `jwt` as a string. Only needed when using JWK validation mode, where the JWT must be signed by the JWK that was initialized with `auth.init()`.

### 3\. auth.session() → jsonb

Retrieves JWT session data. The behavior depends on whether a JWK is defined:

- When JWK is defined:
  - Returns the entire validated JWT payload as JSONB.
  - The JWT must be properly signed and validated.
  - Contains all claims from the JWT (sub, role, etc.).

- When JWK is not defined:
  - Falls back to using the value from PostgREST-compatible `request.jwt.claims` parameter.
  - Returns the claims as JSONB if the parameter is set and contains valid JSON.
  - Returns JSON null if `request.jwt.claims` is not set, is empty, or contains invalid JSON.

This dual behavior allows for flexible session management while maintaining security when JWK is available, and compatibility with PostgREST JWT claims when operating without JWK.

### 4\. auth.user\_id() → text

Returns the user ID associated with the current session. The behavior depends on whether a JWK is defined:

- When JWK is defined:
  - Returns the value from the `"sub"` ("subject") field of the JWT.
  - The JWT must be properly signed and validated.

- When JWK is not defined:
  - Falls back to using the value from the `"sub"` field in the PostgREST-compatible `request.jwt.claims` parameter.
  - Returns NULL if `request.jwt.claims` is not set, is empty, or does not contain a valid string in its `"sub"` field.

This dual behavior allows for flexible authentication scenarios while maintaining security when JWK is available, and compatibility with PostgREST JWT claims when operating without JWK.

License
-------
This project is licensed under the Apache License 2.0. See the LICENSE file for details.

Contact
-------
For issues, questions, or support, please open an issue on the GitHub repository.

### Security
Neon adheres to the [securitytxt.org](https://securitytxt.org/) standard for transparent and efficient security reporting. For details on how to report potential vulnerabilities, please visit our [Security reporting](https://neon.tech/docs/security/security-reporting) page or refer to our [security.txt](https://neon.tech/security.txt) file.

If you have any questions about our security protocols or would like a deeper dive into any aspect, our team is here to help. You can reach us at [security@neon.tech](security@neon.tech).
