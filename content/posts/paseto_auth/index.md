+++
title = 'OTP PASETO auth with gRPC in Rust'
date = '2024-05-13'
readTime = 'true'
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
