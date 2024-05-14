+++
title = 'OTP PASETO Authentication with gRPC in Rust'
date = '2024-05-13'
+++

PASETO is a secure alternative to JWT tokens. It is a token protocol created to patch some of the vulnerabilities of JWT tokens. There are tutorials on how to use PASETO tokens for REST API authentication, but not many on how to use them for gRPC authentication, and even fewer in Rust.

In this post I will show you how to use PASETO tokens for gRPC authentication in Rust. I'll be setting a gRPC server but I will not be covering how to set up a gRPC client. For testing, I'm using Postman, but you can use any gRPC client you prefer.

Here are the dependencies we'll be using:

- Docker compose for bringing up local container instances
- Twilio for sending OTPs
- [`tonic`](https://github.com/hyperium/tonic) for gRPC in Rust
- [`pasetors`](https://github.com/brycx/pasetors) for PASETO token verifiations in Rust
- [`deadpool-postgres`](https://github.com/bikeshedder/deadpool) for persisting refresh tokens
- [`cornucopia`](https://github.com/cornucopia-rs/cornucopia) for compile time SQL checking

**_I'm purposely avoiding `sqlx` because its [performance is extremely poor](https://www.reddit.com/r/rust/comments/1btdrc1/does_sqlx_really_have_more_overhead_in_rust_than/) and its compile time SQL checks are intrusive. Yes I know you can enable offline mode for SQLx but it still requires the use of their migration cli. Overall, the crate is too heavy for my taste._**

## Setting up the gRPC server

Run `cargo new grpc-paseto-server` to create a new Rust project. Add the following dependencies to your `Cargo.toml`:

```toml
[package]
name = "grpc-paseto-server"
version = "0.1.0"
edition = "2021"

[dependencies]
dotenv = "0.15"
config = "0.14"
thiserror = "1.0"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
tokio = { version = "1.37", features = ["rt-multi-thread", "macros", "signal"] }
tonic = { version = "0.11", features = ["default", "tls"] }
prost = "0.12"
tracing = { version = "0.1" }
tracing-subscriber = { version = "0.3", features = ["tracing-log", "fmt"] }
tower = "0.4"
tower-http = { version = "0.4", features = ["full"] }
http = "0.2"
http-body = "1.0"
cornucopia_async = { version = "0.6", features = ["with-serde_json-1"] }
futures = "0.3"
deadpool-postgres = "0.12"
postgres-types = { version = "0.2", features = ["derive"] }
tokio-postgres = { version = "0.7", features = [
    "with-serde_json-1",
    "with-time-0_3",
    "with-uuid-1",
    "with-eui48-1",
] }
eui48 = "1.1"
rust_decimal = { version = "1.35", features = ["db-postgres"] }
uuid = { version = "1.3", features = ["serde", "v4"] }
time = { version = "0.3", features = ["serde"] }
reqwest = { version = "0.12", features = ["json"] }
pasetors = "0.6"
phonenumber = "0.3"
duration-string = { version = "0.3", features = ["serde"] }
anyhow = "1.0"
tonic-types = "0.11"
tonic-middleware = "0.1"
prost-types = "0.12"


[build-dependencies]
tonic-build = { version = "0.11", features = ["prost"] }


[dev-dependencies]
rusty_paserk = "0.3.0"
```

I know I'm cheating here but let me explain some of the core dependencies:

- `dotenv` and `config` for loading environment variables
- `prost` is a requirement for `tonic`
- `tower` and `tower-http` for middlewares
- `tonic-build` for generating gRPC code at compile time (inside `build.rs`)
- `rusty_paserk` for generating PASETO keypairs

### Definiting the ProtoBuf files

Create a new directory called `proto` and add a new file called `auth.proto` with the following content:

```proto
// proto/auth.proto
syntax = "proto3";

package auth;

import "google/protobuf/empty.proto";

// AuthService defines the gRPC service for handling authentication.
service AuthService {
    // RequestOTP is used to send a One Time Password (OTP) to a user's phone number.
    rpc RequestOTP(OTPRequest) returns (google.protobuf.Empty);
    // VerifyOTP is used to verify the OTP sent to the user's phone.
    rpc VerifyOTP(VerifyOTPRequest) returns (TokenResponse);
    // RefreshToken is used to refresh the access token with the refresh token.
    rpc RefreshToken(google.protobuf.Empty) returns (TokenResponse);
    // Provide a valid access token in the Authorization header to logout the user.
    rpc Logout(google.protobuf.Empty) returns (google.protobuf.Empty);
}

// OTPRequest is the request message for requesting an OTP.
message OTPRequest {
    // The user's phone number to which the OTP will be sent.
    string phone_number = 1;
}

// VerifyOTPRequest is the request message for verifying an OTP.
message VerifyOTPRequest {
    // The user's phone number for which the OTP was requested.
    string phone_number = 1;
    // The OTP that was sent to the user's phone number.
    string otp = 2;
}

// TokenResponse is the response message that includes the access token a refresh token.
message TokenResponse {
    // The new access token.
    string access_token = 1;
    // The new refresh token.
    string refresh_token = 2;
}
```

Here, we define a gRPC service called `AuthService` with four RPC methods:

1. `RequestOTP` - Used to send a One Time Password (OTP) to a user's phone number.
2. `VerifyOTP` - Used to verify the OTP sent to the user's phone.
3. `RefreshToken` - Used to refresh the access token with the refresh token once the access token expires.
4. `Logout` - Used to logout the user by invalidating the refresh token. Note, because we're not storing the refresh token in a database, we can't invalidate it. This endpoint is used to invalidate the refresh token, which is essentailly "better than nothing"...

### Generating the gRPC code

To generate the gRPC code, add a new file called `build.rs` to the root of your project with the following content:

```rust
// build.rs
fn main() {
    tonic_build::compile_protos("proto/auth.proto").expect("Failed to compile auth.proto");
}
```

What's nice about tonic is that it generates the gRPC code at compile time for you, so you don't need to worry about running `protoc` manually. Infact, `tonic` does not use the `protoc` binary at all, so you don't need to have it installed on your system.

To test it out, add the following snippet to your `main.rs`:

```rust
// src/main.rs
pub mod auth_pb {
    tonic::include_proto!("auth");
}
```

Now when you run `cargo build`, you should see a warning for not using the imported module, but other than that, that's it!

### Implementing the gRPC auth server

Create a two new file, `grpc/auth.rs` and `grpc.rs` with the following content:

```rust
// src/grpc/auth.rs
pub mod auth_pb {
    tonic::include_proto!("auth");
}

use auth_pb::auth_service_server::AuthService;
use auth_pb::{OtpRequest, TokenResponse, VerifyOtpRequest};
use tonic::{
    transport::server::{TcpConnectInfo, TlsConnectInfo},
    Code, Request, Response, Status,
};
use tonic_types::{ErrorDetails, StatusExt};

pub struct AuthServerImpl {
    pub postgres_pool: deadpool_postgres::Pool,
}

#[tonic::async_trait]
impl AuthService for AuthServerImpl {
    async fn request_otp(&self, request: Request<OtpRequest>) -> Result<Response<()>, Status> {
        unimplemented!()
    }

    async fn verify_otp(
        &self,
        request: Request<VerifyOtpRequest>,
    ) -> Result<Response<TokenResponse>, Status> {
        unimplemented!()
    }

    async fn refresh_token(&self, request: Request<()>) -> Result<Response<TokenResponse>, Status> {
        unimplemented!()
    }

    async fn logout(&self, request: Request<()>) -> Result<Response<()>, Status> {
        unimplemented!()
    }
}
```

Nothing fancy here, we're defining the methods of the `AuthService`. Now, import the `auth` module in `grpc.rs` and the `grpc` module in `main.rs`.

We need an config reader to read values from the environment. Create a new file called `settings.rs` with the following content:

```rust
// src/settings.rs
use duration_string::DurationString;
use serde::Deserialize;

#[derive(Debug, Deserialize)]
pub struct App {
    pub port: String,
    pub loglevel: String,
}

#[derive(Debug, Deserialize)]
pub struct Postgres {
    pub host: String,
    pub port: u16,
    pub user: String,
    pub password: String,
    pub db: String,
}

#[derive(Debug, Deserialize)]
pub struct Twilio {
    pub servicesid: String,
    pub accountsid: String,
    pub authtoken: String,
}

#[derive(Debug, Deserialize)]
pub struct Paseto {
    pub accesssecret: String,
    pub accessexpiry: DurationString,
    pub refreshsecret: String,
    pub refreshexpiry: DurationString,
    pub issuer: String,
}

#[derive(Debug, Deserialize)]
pub struct Settings {
    pub app: App,
    pub postgres: Postgres,
    pub twilio: Twilio,
    pub paseto: Paseto,
}
```

[`DurationString`](https://github.com/Ronniskansing/duration-string) is a string parser that allows us to specify durations like "5m", "5h", or "5d".
The struct values are named without any separator because the `config-rs` crate we're using does not support defininig a fixed depth for nested configs. i.e. an environment variable `A_B_C` will need to be in a struct like this:

```rust
pub struct B {
    pub c: String,
}
pub struct A {
    pub b: B,
}
pub struct Settings {
    pub a: A,
}
```

We don't want this so I'm going to be defining the environment variables with only a single separator. Hence the `accesssecret` field inside the `Paseto` struct will be from an environment variable named `PASETO_ACCESSSECRET`.

Now we can define a `.env` file with the following content:

```env
# .env
APP_PORT=8000
APP_LOGLEVEL=INFO

POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=dev
POSTGRES_PASSWORD=dev_password123
POSTGRES_DB=dev_db

TWILIO_SERVICESID=<YOUR_TWILIO_SERVICESID>
TWILIO_ACCOUNTSID=<YOUR_TWILIO_ACCOUNTSID>
TWILIO_AUTHTOKEN=<YOUR_TWILIO_AUTHTOKEN>

PASETO_ACCESSSECRET=<YOUR_PASETO_ACCESSSECRET>
PASETO_ACCESSEXPIRY=15m
PASETO_REFRESHSECRET=<YOUR_PASETO_REFRESHSECRET>
PASETO_REFRESHEXPIRY=3h
PASETO_ISSUER=grpc://<YOUR_DOMAIN>
```

You can replace the contents inside of `<...>` with your own values for now. We'll need them down the road but don't worry about its correctness.

Now we can update our `main.rs` to read the settings from the environment and start the gRPC server:

```rust
// src/main.rs
mod grpc;
mod settings;

use config::{Config, Environment};
use deadpool_postgres::Runtime;
use dotenv::dotenv;
use grpc::auth::auth_pb::auth_service_server::AuthServiceServer;
use grpc::auth::AuthServerImpl;
use settings::Settings;
use std::iter::once;
use tokio::signal::unix::{signal, SignalKind};
use tokio_postgres::NoTls;
use tonic::transport::Server;
use tonic_middleware::InterceptorFor;
use tower_http::{
    sensitive_headers::SetSensitiveRequestHeadersLayer,
    trace::{DefaultMakeSpan, DefaultOnRequest, DefaultOnResponse, TraceLayer},
    LatencyUnit,
};

#[tokio::main]
async fn main() -> Result<(), anyhow::Error> {
    // Load env vars from .env file
    dotenv().ok();

    // Build configs from the environment
    let settings: Settings = Config::builder()
        .add_source(Environment::default().separator("_"))
        .build()?
        .try_deserialize()?;

    // Read tracing log level from settings
    let log_level = settings
        .app
        .loglevel
        .parse()
        .unwrap_or(tracing::Level::INFO); // default to INFO
    tracing_subscriber::fmt().with_max_level(log_level).init();

    let postgres_pool = deadpool_postgres::Config {
        host: Some(settings.postgres.host),
        port: Some(settings.postgres.port),
        user: Some(settings.postgres.user),
        password: Some(settings.postgres.password),
        dbname: Some(settings.postgres.db),
        ..Default::default()
    }
    .create_pool(Some(Runtime::Tokio1), NoTls)?;

    // Create a signal receiver for SIGINT and SIGTERM for graceful shutdown
    let mut sigint = signal(SignalKind::interrupt())?;
    let mut sigterm = signal(SignalKind::terminate())?;

    let auth = AuthServerImpl {
        postgres_pool: postgres_pool.clone(),
    };

    let addr = format!("[::]:{}", settings.app.port).parse()?;
    tracing::info!("Listening on {}", addr);
    Server::builder()
        // Mark the `Authorization` request header as sensitive so it doesn't show in logs
        .layer(SetSensitiveRequestHeadersLayer::new(once(
            http::header::AUTHORIZATION,
        )))
        .layer(
            // Request and response logging middleware
            TraceLayer::new_for_http()
                .make_span_with(
                    DefaultMakeSpan::new()
                        .level(log_level)
                        .include_headers(true),
                )
                .on_request(DefaultOnRequest::new().level(log_level))
                .on_response(
                    DefaultOnResponse::new()
                        .level(log_level)
                        .include_headers(true)
                        .latency_unit(LatencyUnit::Millis),
                ),
        )
        .trace_fn(|_| tracing::info_span!("grpc-paseto-server"))
        .add_service(AuthServiceServer::new(auth))
        .serve_with_shutdown(addr, async move {
            // Listen for SIGINT and SIGTERM signals
            tokio::select! {
                _ = sigint.recv() => tracing::warn!("[SIGINT] Shutting down..."),
                _ = sigterm.recv() => tracing::warn!("[SIGTERM] Shutting down...")
            }
        })
        .await?;

    Ok(())
}
```

We did a few things here:

1. We read the some environment variables from the `.env` file using the call to `dotenv().ok()`. Then we used `config::Config::builder()` to read those environment values into our `Settings` struct.
2. We also set up the logging level based on the `APP_LOGLEVEL` environment variable.
3. We then created a `deadpool_postgres::Config` struct to create a connection pool to our PostgreSQL database.
4. We then created an instance of our `AuthServerImpl` struct and passed it to the `AuthServiceServer::new` function to create a gRPC server. We then used `tonic::transport::Server::builder()` to create a new gRPC server and added some middleware to it.
5. We then called `serve_with_shutdown` to start the server and listen for the `SIGINT` and `SIGTERM` signals, which is definitely a overkill for now but it's good to have it in place.

### Building the docker image

Now lets move on to building the docker image for our application. We'll be using `docker compose` to facilitate the the container running process. In production, you'll need to replace the contains inside of the `compose.yml` file your k8s deployment files and keep the secrets we defined in the `.env` file as k8s secrets. instead of an environment variable. How to do this is beyond the scope of this tutorial.

Firstly, lets create a `Dockerfile` in the root of our project:

```Dockerfile
# Dockerfile
FROM rust:latest as builder

# Install protobuf compiler
RUN apt update && \
    apt install -y protobuf-compiler libprotobuf-dev

# Add GitHub's SSH host keys to the known hosts
RUN mkdir -p /root/.ssh/ && \
    ssh-keyscan github.com >> /root/.ssh/known_hosts

WORKDIR /builder
COPY . .

# Build with the mounted SSH key as it pulls the git submodules
# inside the build script
RUN --mount=type=ssh cargo build --release


FROM debian:bookworm-slim

RUN apt update && \
    apt install -y ca-certificates

WORKDIR /app
COPY --from=builder /builder/target/release/grpc-paseto-server .
ENTRYPOINT ["/app/grpc-paseto-server"]
```

I like have extremely tiny Docker images, but to do that you'll need to use an `alpine` based image, which would require you to do **cross-compilation**, which is a bit more complex. So I'm using a `debian` based image here. This image turns out to be only about 100MB, but you can check out how to do cross-compilation with `alpine` based images [here](https://kerkour.com/rust-cross-compilation) if you want to go that route.

Next, we'll create a `compose.yml` file in the root of our project:

```yml
# compose.yml
services:
  grpc-paseto-server:
    image: grpc-paseto-server
    build:
      context: .
      dockerfile: Dockerfile
      ssh:
        - default
    environment:
      - POSTGRES_HOST=my-postgres
    ports:
      - "8000:8000"
    env_file:
      - ./.env
    depends_on:
      - my-postgres

  my-postgres:
    image: postgres:latest
    container_name: my-postgres
    ports:
      - "5432:5432"
    env_file:
      - ./.env
    volumes:
      - ./schema.sql:/docker-entrypoint-initdb.d/init.sql:readonly
```

A few things to note here:

- We're defining the `ssh` field under `build` to mount the SSH key to the container. This allows us to fetch from private repositories just in case. Hence the `--mount=type=ssh` flag when we build the image.
- We're using the `.env` file to set the environment variables for the `grpc-paseto-server` and `my-postgres` services. This tells docker compose to read the environment variables from the `.env` file and set them as environment variables for the services.
- We're then overwriting the `POSTGRES_HOST` environment variable from the `.env` file becuase this hostname appears as `my-postgres` to the `grpc-paseto-server` container when we run docker compose.

### Initializing the Postgres database

Notice how we're mounting the `schema.sql` file to the `my-postgres` container to initialize the database with the schema from the previous `compose.yml` file, but we haven't created the `schema.sql` file yet. Let's do that now.

Create a `schema.sql` file in the root of your project. This file will later also be used by `cornucopia`:

```sql
-- schema.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TYPE token_type AS ENUM ('refresh', 'access');
CREATE TABLE
    "tokens" (
        id UUID NOT NULL PRIMARY KEY DEFAULT (uuid_generate_v4()),
        user_id UUID NOT NULL DEFAULT (uuid_generate_v4()),
        token_type token_type NOT NULL DEFAULT 'refresh',
        is_revoked BOOLEAN NOT NULL DEFAULT FALSE
    );
CREATE INDEX tokens_user_id_idx ON tokens (user_id);

CREATE TYPE onboarding_status AS ENUM ('new', 'completed');
CREATE TYPE user_role AS ENUM ('user', 'admin');
CREATE TABLE
    "users" (
        id UUID NOT NULL PRIMARY KEY DEFAULT (uuid_generate_v4()),
        onboarding onboarding_status NOT NULL DEFAULT 'new',
        role user_role NOT NULL DEFAULT 'user',
        created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
        updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
        name VARCHAR(100),
        phone_number VARCHAR(20) UNIQUE
    );
CREATE INDEX user_phone_number_idx ON users (phone_number);
```

Notice how the `user_id` gets a default value of `uuid_generate_v4()`. This is because in the OTP workflow, we'll be creating a new user and generating a new token for them, even if they don't exist in the database yet. You are then responsible for creating other endpoints that handles the creation of an user when they have a valid token but no user profile. A bit more convoluted, but leads to a nicer user experience.

The `users` table is something you can manipulate to your liking, I've just added a `name` and `phone_number` field to it. You can add more fields as you see fit.

Now the gRPC server should be setup, although none of the endpoints are implemented yet. We'll do that in the next section.

## Implementing the gRPC endpoints

We previously stubbed out our auth methods with `unimplemented!()`. Now we'll implement them, but before we can do that, we need a Twilio client to call the Twilio Verify API. We'll create a `TwilioClient` struct that will handle the communication with the Twilio Verify API. I'm not going to cover setting up Twilio Verify here, but you can check out the [Twilio Verify API documentation](https://www.twilio.com/docs/verify/api) to get started.

### Setting up the Twilio client

Create a `vendors.rs` file in the `src` directory with the following:

```rust
// src/vendors.rs
use serde::{Deserialize, Serialize};

#[derive(Clone)]
pub struct TwilioClient {
    http_client: reqwest::Client,
    twilio_account_sid: String,
    twilio_auth_token: String,
    twilio_verify_v2_url: String,
    twilio_verify_check_v2_url: String,
}

impl TwilioClient {
    pub fn new(
        twilio_service_sid: String,
        twilio_account_sid: String,
        twilio_auth_token: String,
    ) -> Self {
        Self {
            http_client: reqwest::Client::new(),
            twilio_account_sid,
            twilio_auth_token,
            twilio_verify_v2_url: format!(
                "https://verify.twilio.com/v2/Services/{}/Verifications",
                twilio_service_sid
            ),
            twilio_verify_check_v2_url: format!(
                "https://verify.twilio.com/v2/Services/{}/VerificationCheck",
                twilio_service_sid
            ),
        }
    }

    pub async fn request_otp(
        &self,
        to: &str,
        channel: VerifyChannel,
    ) -> Result<reqwest::Response, reqwest::Error> {
        self.http_client
            .post(&self.twilio_verify_v2_url)
            .basic_auth(&self.twilio_account_sid, Some(&self.twilio_auth_token))
            .form(&VerifyRequest { to, channel })
            .send()
            .await
    }

    pub async fn verify_otp(
        &self,
        to: &str,
        code: &str,
    ) -> Result<reqwest::Response, reqwest::Error> {
        self.http_client
            .post(&self.twilio_verify_check_v2_url)
            .basic_auth(&self.twilio_account_sid, Some(&self.twilio_auth_token))
            .form(&VerifyCheckRequest { to, code })
            .send()
            .await
    }
}

#[derive(Copy, Clone, Debug, PartialEq, Eq, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum VerifyChannel {
    Sms,
}

#[derive(Serialize)]
#[serde(rename_all = "PascalCase")]
struct VerifyRequest<'a> {
    to: &'a str,
    channel: VerifyChannel,
}

#[derive(Serialize)]
#[serde(rename_all = "PascalCase")]
struct VerifyCheckRequest<'a> {
    to: &'a str,
    code: &'a str,
}

#[derive(Deserialize, PartialEq, Eq)]
#[serde(rename_all = "snake_case")]
pub enum VerifyStatus {
    Approved,
    Pending,
    Canceled,
}

#[derive(Deserialize)]
#[serde(rename_all = "snake_case")]
pub struct VerifyResponse {
    pub to: String,
    pub status: VerifyStatus,
}
```

We defined two methods on the `TwilioClient` struct: `request_otp` and `verify_otp`. The `request_otp` method sends an OTP to a phone number, while the `verify_otp` method verifies the OTP code sent to the phone number. All of this is documented in the [Twilio Verify API documentation](https://www.twilio.com/docs/verify/api). Please refer to it if you need, for example, rate limiting or other features.

Notice that I've left out the validation of the Twilio response here. I think it's good practice to separate out the implementation of the Twilio client from the validation of the response.

### Implementing the gRPC endpoints

First we'll need to update the `AuthServerImpl` struct with some extra fields and methods:

```rust
// src/grpc/auth.rs
pub mod auth_pb {
    tonic::include_proto!("auth");
}

use crate::cornucopia::queries::tokens;
use crate::cornucopia::queries::users;
use crate::vendors::{TwilioClient, VerifyChannel, VerifyResponse, VerifyStatus};
use anyhow::Context;
use auth_pb::auth_service_server::AuthService;
use auth_pb::{OtpRequest, TokenResponse, VerifyOtpRequest};
use pasetors::keys::AsymmetricKeyPair;
use pasetors::{
    claims::{Claims, ClaimsValidationRules},
    token::UntrustedToken,
    Public,
};
use pasetors::{public, version4::V4};
use phonenumber::{Mode, PhoneNumber};
use tonic::{
    transport::server::{TcpConnectInfo, TlsConnectInfo},
    Code, Request, Response, Status,
};
use tonic_types::{ErrorDetails, StatusExt};

const IMPLICT_ASSERTION: &[u8] = b"grpc-paseto-server";
const UNKNOWN_IP_DEFAULT: &str = "UNKNOWN_IP";

pub struct AuthServerImpl {
    pub postgres_pool: deadpool_postgres::Pool,
    pub twilio_client: TwilioClient,
    pub token_issuer: String,
    pub access_token_key_pair: AsymmetricKeyPair<V4>,
    pub access_token_expiration: std::time::Duration,
    pub refresh_token_key_pair: AsymmetricKeyPair<V4>,
    pub refresh_token_expiration: std::time::Duration,
}

impl AuthServerImpl {
    fn gen_token_resp(
        &self,
        issuer: &str,
        subject: &str,
        audience: &str,
        token_identifier: &str,
        phone_number: &str,
    ) -> Result<TokenResponse, anyhow::Error> {
        let mut access_claim = Claims::new_expires_in(&self.access_token_expiration)?;
        access_claim.issuer(issuer).context("Access claim issuer")?;
        access_claim
            .subject(subject)
            .context("Access claim subject")?;
        access_claim
            .audience(audience)
            .context("Access claim audience")?;
        access_claim
            .add_additional("phone_number", phone_number)
            .context("Acces claim phone_number")?;
        let access_token = public::sign(
            &self.access_token_key_pair.secret,
            &access_claim,
            None,
            Some(IMPLICT_ASSERTION),
        )
        .context("Access claim signing")?;

        let mut refresh_claim = Claims::new_expires_in(&self.refresh_token_expiration)?;
        refresh_claim
            .issuer(issuer)
            .context("Refresh claim issuer")?;
        refresh_claim
            .subject(subject)
            .context("Refresh claim subject")?;
        refresh_claim
            .token_identifier(token_identifier)
            .context("Refresh claim token_identifier")?;
        refresh_claim
            .add_additional("phone_number", phone_number)
            .context("Refresh claim phone_number")?;
        let refresh_token = public::sign(
            &self.refresh_token_key_pair.secret,
            &refresh_claim,
            None,
            Some(IMPLICT_ASSERTION),
        )
        .context("Refresh claim signing")?;

        Ok(TokenResponse {
            access_token,
            refresh_token,
        })
    }
}

// ...
```

I'm cheating again by adding all of the imports at once to save time. The implementation here includes a `gen_token_resp` method, which generates the access and refresh tokens. We also added some extra fields to the `AuthServerImpl` struct to hold the token issuer, key pairs, and expiration times for the access and refresh tokens.

### The `request_otp` method

```rust
// src/grpc/auth.rs
//
// ... stuff we've already covered from previous sections

async fn request_otp(&self, request: Request<OtpRequest>) -> Result<Response<()>, Status> {
    // Parse phone numer from request. Makes sure it's valid.
    let phonenumber = match request.into_inner().phone_number.parse::<PhoneNumber>() {
        Ok(pn) => pn.format().mode(Mode::E164).to_string(),
        Err(_) => {
            return Err(Status::with_error_details(
                Code::InvalidArgument,
                "Invalid phone number",
                ErrorDetails::with_bad_request_violation(
                    "phone_number",
                    "Need to be in E164 format. Example: +14155552671",
                ),
            ));
        }
    };

    // Request OTP from Twilio using the phone number
    match self
        .twilio_client
        .request_otp(&phonenumber, VerifyChannel::Sms)
        .await
    {
        Ok(resp) => match resp.status() {
            reqwest::StatusCode::CREATED => Ok(Response::new(())),
            reqwest::StatusCode::TOO_MANY_REQUESTS => Err(Status::with_error_details(
                Code::InvalidArgument,
                "Too many requests",
                ErrorDetails::with_quota_failure_violation(
                    "phone_number",
                    "Too many requests in a short period of time",
                ),
            )),
            _ => {
                tracing::warn!("Twilio OTP verify response unexpected response: {:?}", resp);
                Err(Status::unknown("Something happened with Twilio"))
            }
        },
        Err(err) => {
            tracing::error!("Twilio OTP verify request error: {:?}", err);
            Err(Status::internal("Idk wtf happened"))
        }
    }
}
```

The `request_otp` method sends an OTP to a phone number. It first validates the phone number and then sends the OTP using the Twilio client. If the OTP is sent successfully, the method returns an empty response. If there's an error, it returns a `Status` with an error message.

### The `verify_otp` method

```rust
// src/grpc/auth.rs
//
// ... stuff we've already covered from previous sections

async fn verify_otp(
    &self,
    request: Request<VerifyOtpRequest>,
) -> Result<Response<TokenResponse>, Status> {
    let req_ip = request
        .remote_addr()
        .map(|addr| addr.to_string())
        .unwrap_or(UNKNOWN_IP_DEFAULT.to_owned());
    let inner = request.into_inner();
    let phonenumber = match inner.phone_number.parse::<PhoneNumber>() {
        Ok(pn) => pn.format().mode(Mode::E164).to_string(),
        Err(_) => {
            return Err(Status::with_error_details(
                Code::InvalidArgument,
                "Invalid phone number",
                ErrorDetails::with_bad_request_violation(
                    "phone_number",
                    "Need to be in E164 format. Example: +14155552671",
                ),
            ));
        }
    };
    let code = inner.otp;

    match self.twilio_client.verify_otp(&phonenumber, &code).await {
        Ok(resp) => match resp.status() {
            reqwest::StatusCode::ACCEPTED | reqwest::StatusCode::OK => {
                // Check if the OTP code is approved
                let twilio_verify_check_resp_json =
                    resp.json::<VerifyResponse>().await.map_err(|e| {
                        tracing::error!("Failed deserializing Twilio response: {}", e);
                        Status::internal("Internal server error")
                    })?;

                match twilio_verify_check_resp_json.status {
                    VerifyStatus::Pending => {
                        return Err(Status::with_error_details(
                            Code::InvalidArgument,
                            "Invalid OTP code",
                            ErrorDetails::with_error_info(
                                "INVALID_OTP_CODE",
                                "AUTHENTICATION",
                                [],
                            ),
                        ));
                    }
                    VerifyStatus::Canceled => {
                        tracing::warn!("Twilio verifcation check was somehow cancelled...");
                        return Err(Status::with_error_details(
                            Code::Cancelled,
                            "OPT verification canceled, request a new one",
                            ErrorDetails::with_error_info(
                                "TWILIO_VERIFY_CANCELED",
                                "AUTHENTICATION",
                                [],
                            ),
                        ));
                    }
                    VerifyStatus::Approved => {
                        // Correct code! Continue...
                    }
                }

                // Verified, generate a token pair
                // Get user id by phone number if the user exists, else create one.
                // User profile is responsible for creating the user if it does not exist.
                let user_id = users::get_user_id_by_phone_number()
                    .bind(
                        &self.postgres_pool.get().await.map_err(|e| {
                            tracing::error!("Failed getting db connection: {}", e);
                            Status::internal("Internal database error")
                        })?,
                        &phonenumber,
                    )
                    .opt()
                    .await
                    .map_err(|e| {
                        tracing::error!("Failed getting UserID by phone number: {}", e);
                        Status::internal("Internal database error")
                    })?
                    .unwrap_or(uuid::Uuid::new_v4());
                // Make a new token id
                let token_id = uuid::Uuid::new_v4();

                match self.gen_token_resp(
                    &self.token_issuer,
                    &user_id.to_string(),
                    &req_ip,
                    &token_id.to_string(),
                    &phonenumber,
                ) {
                    Ok(token_resp) => {
                        // Store the refresh token in the database
                        {
                            let postgres_client =
                                self.postgres_pool.get().await.map_err(|e| {
                                    tracing::error!("Failed getting db connection: {}", e);
                                    Status::internal("Internal database error")
                                })?;

                            // Revoke any existing tokens if there are any
                            tokens::hard_revoke_tokens_by_user_id()
                                .bind(&postgres_client, &user_id)
                                .await
                                .map_err(|e| {
                                    tracing::error!("Failed revoking token: {}", e);
                                    Status::internal("Internal database error")
                                })?;

                            // Insert the new token with user_id
                            tokens::insert_token_with_user_id()
                                .bind(&postgres_client, &token_id, &user_id)
                                .await
                                .map_err(|e| {
                                    tracing::error!("Failed storing token: {}", e);
                                    Status::internal("Internal database error")
                                })?;
                        }

                        Ok(Response::new(token_resp))
                    }
                    Err(err) => {
                        tracing::error!("Error generating tokens: {:?}", err);
                        Err(Status::internal("Idk wtf happened"))
                    }
                }
            }
            reqwest::StatusCode::NOT_FOUND => Err(Status::not_found(
                "Verification already completed, expired, or reached max attempts",
            )),
            _ => {
                tracing::warn!(
                    "Twilio OTP verify_check response unexpected response: {:?}",
                    resp
                );
                Err(Status::unknown("Something happened with Twilio"))
            }
        },
        Err(err) => {
            tracing::error!("Twilio OTP verify_check request error: {:?}", err);
            Err(Status::internal("Idk wtf happened"))
        }
    }
}
```

The `verify_otp` method first validates the phone number and OTP code. Then it sends the OTP code to Twilio for verification. If the OTP code is correct, the method generates a token pair and returns it. If there's an error, it returns a `Status` with an error message.

Notice that we've added a few Postgres queries that seem to have came out of nowhere. We'll cover them in in a bit.

### The `refresh_token` method

The `refresh_token` method is responsible for refreshing the access token when it expires. This is a common auth pattern and I won't be covering it in detail, but know that in this workflow, both the refresh and access tokens are stored on the client but only the refresh token is stored on the server. See the previous section for how that was first created.

```rust
// src/grpc/auth.rs
//
// ... stuff we've already covered from previous sections

async fn refresh_token(&self, request: Request<()>) -> Result<Response<TokenResponse>, Status> {
    match request.metadata().get("authorization").map(|v| v.to_str()) {
        Some(Ok(raw)) => {
            let token_str = raw.strip_prefix("Bearer ").ok_or(Status::invalid_argument(
                "Authorization header missing 'Bearer ' prefix",
            ))?;

            let untrusted_token =
                UntrustedToken::<Public, V4>::try_from(token_str).map_err(|e| {
                    Status::invalid_argument(format!("Invalid token encoding: {}", e))
                })?;
            let untrusted_claim = Claims::from_bytes(untrusted_token.untrusted_payload())
                .map_err(|e| {
                    Status::invalid_argument(format!("Invalid token payload: {}", e))
                })?;
            let token_id_str = untrusted_claim
                .get_claim("jti")
                .ok_or(Status::invalid_argument(
                    "Token identifier not found in token payload",
                ))?
                .as_str()
                .ok_or(Status::invalid_argument(
                    "Token identifier must be a string",
                ))?;

            let mut validation_rules = ClaimsValidationRules::new();
            validation_rules.validate_issuer_with(&self.token_issuer);
            validation_rules.validate_token_identifier_with(token_id_str);
            // 'sub' == the user_id is validated below, not here.

            let trusted_token = public::verify(
                &self.refresh_token_key_pair.public,
                &untrusted_token,
                &validation_rules,
                None,
                Some(IMPLICT_ASSERTION),
            )
            .map_err(|e| {
                tracing::warn!("Refresh token verification failed: {:?}", e);
                Status::with_error_details(
                    Code::Unauthenticated,
                    "Invalid refresh token",
                    ErrorDetails::with_error_info(
                        "REFRESH_TOKEN_VERIFICATION_FAILURE",
                        "AUTHENTICATION",
                        [],
                    ),
                )
            })?;

            let token_id = uuid::Uuid::parse_str(token_id_str).map_err(|e| {
                tracing::error!("Failed parsing token id after verification: {}", e);
                Status::internal("Internal server in a weird state")
            })?;

            let payload_claims = trusted_token
                .payload_claims()
                .ok_or(Status::invalid_argument("Payload not found"))?;

            let user_id_str = payload_claims
                .get_claim("sub")
                .ok_or(Status::invalid_argument("Subject not found"))?
                .as_str()
                .ok_or(Status::invalid_argument("Subject must be a string"))?;
            let user_id = uuid::Uuid::parse_str(user_id_str).map_err(|e| {
                tracing::error!("Failed parsing user id after verification: {}", e);
                Status::internal("Internal server in a weird state")
            })?;

            let phone_number_str = payload_claims
                .get_claim("phone_number")
                .ok_or(Status::invalid_argument("Phone number not found"))?
                .as_str()
                .ok_or(Status::invalid_argument("Phone number must be a string"))?;

            let valid_token_cnt = {
                let postgres_client = self.postgres_pool.get().await.map_err(|e| {
                    tracing::error!("Failed getting db connection: {}", e);
                    Status::internal("Internal database error")
                })?;

                // Check the refresh token is in the database
                let valid_token_cnt = tokens::get_valid_token_count()
                    .bind(&postgres_client, &token_id, &user_id)
                    .one()
                    .await
                    .map_err(|e| {
                        tracing::error!("Failed getting valid token count: {}", e);
                        Status::internal("Internal database error")
                    })?;

                // Revoke any existing tokens
                // this needs to happen regardless of whether if the token is in the database
                tokens::hard_revoke_tokens_by_user_id()
                    .bind(&postgres_client, &user_id)
                    .await
                    .map_err(|e| {
                        tracing::error!("Failed revoking token: {}", e);
                        Status::internal("Internal database error")
                    })?;

                valid_token_cnt
            };

            match valid_token_cnt {
                0 => {
                    tracing::warn!(
                        "Refresh token verified but not found in database, potential attack!"
                    );

                    Err(Status::with_error_details(
                        Code::Unauthenticated,
                        "Invalid refresh token",
                        ErrorDetails::with_error_info(
                            "REFRESH_TOKEN_DOES_NOT_EXIST",
                            "AUTHENTICATION",
                            [],
                        ),
                    ))
                }
                1 => {
                    let new_refresh_token_id = uuid::Uuid::new_v4();
                    let req_ip = request
                        .remote_addr()
                        .map(|addr| addr.to_string())
                        .unwrap_or(UNKNOWN_IP_DEFAULT.to_owned());
                    match self.gen_token_resp(
                        &self.token_issuer,
                        user_id_str,
                        &req_ip,
                        &new_refresh_token_id.to_string(),
                        phone_number_str,
                    ) {
                        Ok(token_resp) => {
                            // Store the new refresh token in the database
                            tokens::insert_token_with_user_id()
                                .bind(
                                    &self.postgres_pool.get().await.map_err(|e| {
                                        tracing::error!("Failed getting db connection: {}", e);
                                        Status::internal("Internal database error")
                                    })?,
                                    &new_refresh_token_id,
                                    &user_id,
                                )
                                .await
                                .map_err(|e| {
                                    tracing::error!("Failed storing token: {}", e);
                                    Status::internal("Internal database error")
                                })?;

                            Ok(Response::new(token_resp))
                        }
                        Err(err) => {
                            tracing::error!("Error generating tokens: {:?}", err);
                            Err(Status::internal("Idk wtf happened"))
                        }
                    }
                }
                _ => {
                    tracing::error!("Valid token count more than 1");
                    Err(Status::internal("Internal server in a weird state"))
                }
            }
        }
        _ => Err(Status::unauthenticated(
            "Invalid or missing authorization header",
        )),
    }
}
```

Here, we first validate the refresh token using `pasetors`. If the refresh token is valid, we invalidate the previous refresh token stored in Postgres and store and send new ones to the client.

### The `logout` method

```rust
// src/grpc/auth.rs
//
// ... stuff we've already covered from previous sections

async fn logout(&self, request: Request<()>) -> Result<Response<()>, Status> {
    match request.metadata().get("authorization").map(|v| v.to_str()) {
        Some(Ok(raw)) => {
            let token_str = raw.strip_prefix("Bearer ").ok_or(Status::invalid_argument(
                "Authorization header missing 'Bearer ' prefix",
            ))?;

            let untrusted_token =
                UntrustedToken::<Public, V4>::try_from(token_str).map_err(|e| {
                    Status::invalid_argument(format!("Invalid token encoding: {}", e))
                })?;

            let mut validation_rules = ClaimsValidationRules::new();
            validation_rules.validate_issuer_with(&self.token_issuer);
            validation_rules.validate_audience_with(
                &request
                    .remote_addr()
                    .map(|addr| addr.to_string())
                    .unwrap_or(UNKNOWN_IP_DEFAULT.to_owned()),
            );

            match public::verify(
                &self.access_token_key_pair.public,
                &untrusted_token,
                &validation_rules,
                None,
                Some(IMPLICT_ASSERTION),
            ) {
                Ok(trusted_token) => {
                    let user_id_str = trusted_token
                        .payload_claims()
                        .ok_or(Status::invalid_argument("Payload not found"))?
                        .get_claim("sub")
                        .ok_or(Status::invalid_argument("Subject not found"))?
                        .as_str()
                        .ok_or(Status::invalid_argument("Subject must be a string"))?;
                    let user_id = uuid::Uuid::parse_str(user_id_str).map_err(|e| {
                        tracing::error!("Failed parsing user id after verification: {}", e);
                        Status::internal("Internal server in a weird state")
                    })?;

                    tokens::hard_revoke_tokens_by_user_id()
                        .bind(
                            &self.postgres_pool.get().await.map_err(|e| {
                                tracing::error!("Failed getting db connection: {}", e);
                                Status::internal("Internal database error")
                            })?,
                            &user_id,
                        )
                        .await
                        .map_err(|e| {
                            tracing::error!("Failed revoking token: {}", e);
                            Status::internal("Internal database error")
                        })?;

                    Ok(Response::new(()))
                }
                Err(err) => match err {
                    pasetors::errors::Error::ClaimValidation => {
                        return Err(Status::with_error_details(
                            Code::Unauthenticated,
                            "Invalid access token",
                            ErrorDetails::with_error_info(
                                "ACCESS_TOKEN_CLAIM_VALIDATION_FAILURE",
                                "AUTHENTICATION",
                                [(
                                    "help".to_owned(),
                                    "Access token possibiliy expired".to_owned(),
                                )],
                            ),
                        ));
                    }
                    _ => {
                        tracing::warn!("Access token verification failed: {:?}", err);
                        return Err(Status::with_error_details(
                            Code::Unauthenticated,
                            "Invalid access token",
                            ErrorDetails::with_error_info(
                                "ACCESS_TOKEN_VERIFICATION_FAILURE",
                                "AUTHENTICATION",
                                [],
                            ),
                        ));
                    }
                },
            }
        }
        _ => {
            return Err(Status::unauthenticated(
                "Invalid or missing authorization header",
            ))
        }
    }
}
```

In the typical token based authentication flow, the `logout` method is a an operation that invalidates the refresh token. The access token is kept alive because the server simply doesn't have the means to revoke it (because it's not stored anywhere on the server).

### Generating SQL queries with `cornucopia`

`cornucopia` is a crate for generating type-checked Rust interfaces from your PostgreSQL queries. It's an alternative approach to using an ORM like `diesel` or compile time checks like `sqlx`, which requires a running database.

Of couse, the downside is that these Rust interfaces are still generated by running the queries against a live database, but you only need it once when you generate the interfaces.

There's two ways to use `cornucopia`:

1. Using the `cornucopia` CLI tool
2. Using `cornucopia` inside `build.rs`

For both methods, you can either specify a connection string to your running database or a `schema.sql` file that contains your Postgres schemas. Hence why we created the `schema.sql` file under root, the default location where cornucopia looks for the schema file.

You define your queries in a separate directory contain `.sql` files under `queries/` in the root of your project (which you can change). You can have multiple `.sql` files in the `queries/` directory and each one will be used to crate a separate module in the generated code.

For this purpose, I've created two queries in the `queries/` directory:

First is the `tokens.sql` file:

```sql
--! soft_revoke_tokens_by_user_id
UPDATE tokens
SET is_revoked = TRUE
WHERE user_id = :user_id;

--! hard_revoke_tokens_by_user_id
DELETE FROM tokens
WHERE user_id = :user_id;

--! insert_token_with_user_id
INSERT INTO tokens (id, user_id)
VALUES (:id, :user_id);

--! get_valid_token_count
SELECT COUNT(*) FROM tokens
WHERE id = :id
AND user_id = :user_id
AND is_revoked = FALSE;
```

And the second is the `users.sql` file:

```sql
--! get_user_by_id : (name?, phone_number?)
SELECT *
FROM users
WHERE id = :id;

--! get_user_id_by_phone_number
SELECT id
FROM users
WHERE phone_number = :phone_number;

--! create_user_by_id_with_phone_number (phone_number?) : (name?, phone_number?)
INSERT INTO users (id, phone_number)
VALUES (:id, :phone_number)
RETURNING *;
```

Notice the `--!` syntax in the SQL queries. This is a special syntax that `cornucopia` uses to parse the queries and generate the Rust interfaces. You can find more about `cornucopia` in their [documentation](https://cornucopia-rs.netlify.app/book/index.html).

I'm using the second method of generation where I'm generating the files inside of `build.rs`:

```rust
// build.rs
use cornucopia::{CodegenSettings, Error};

fn main() -> Result<(), Error> {
    // Compile the proto file
    tonic_build::compile_protos("proto/auth.proto").expect("Failed to compile auth.proto");

    // Generate the SQL interfaces
    let queries_path = "queries";
    let schema_file = "schema.sql";
    let destination = "src/cornucopia.rs";
    let settings = CodegenSettings {
        gen_async: true,
        gen_sync: false,
        derive_ser: true,
    };

    println!("cargo:rerun-if-changed={queries_path}");
    println!("cargo:rerun-if-changed={schema_file}");
    cornucopia::generate_managed(
        queries_path,
        &[schema_file],
        Some(destination),
        false,
        settings,
    )?;

    Ok(())
}
```

The way this works is a little funky. It essentially does exactly what the CLI tool will do by spinning up a Postgres Docker container and running your `shema.sql` SQL against it. The generated Rust interfaces are then written to a `src/cornucopia.rs` file. For this to work, you obvious need to have docker installed, but you may also run into some issues with the Docker daemon not being accessible. If that happens, just run your own Postgres container and pass the connection string to the `cornucopia` CLI instead.
