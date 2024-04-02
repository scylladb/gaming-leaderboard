# Quick start: Rust 

In this tutorial you'll build a Gaming Leaderboard to store runs from a rhythm game. 

## 1. Setup the Enviroment

Let's download Rust and the dependencies needed for this project. 

### 1.1 Downloading Rust and Dependencies

If you don't have rust installed in your machine yet, run the command below and it will install Rust and some other helpful tools (such as Cargo).
```
$ curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### 1.2 Starting the project

Now with the Rust and Cargo installed, just create a new project using this command:

```sh
cargo new leaderboard-rust
```

### 1.3 Setting up the .env 

We're gonna be using sensitive credentials on our project, to connect into ScyllaDB Cluster, so let's prepare an `.env` file to handle that. Create a new env in the root folder, next to `cargo.toml` and replace to your cluster credentials:

```
# App Config
APP_NAME="Gaming Leaderboard"
APP_VERSION="0.0.1"
APP_URL="0.0.0.0"
APP_PORT="8000"

# Database Config
SCYLLA_NODES="node-0.clusters.scylla.cloud,node-1.clusters.scylla.cloud,node-2.clusters.scylla.cloud"
SCYLLA_USERNAME="scylla"
SCYLLA_PASSWORD="your-password"
SCYLLA_CACHED_QUERIES="15"
SCYLLA_KEYSPACE="leaderboard"

```

### 1.4 Setting the project dependencies



Let's do a quick change into our `cargo.toml` and add our project dependencies. 

```toml
[package]
name = "leaderboard-rust"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4.5.1"
charybdis = "0.4.2"
chrono = "0.4.34"
dotenvy = "0.15.7"
scylla = { version = "0.12.0", features = ["time", "chrono"] }
serde = { version = "1.0.197", features = ["derive"] }
serde_json = "1.0.113"
thiserror = "1.0.56"
uuid = { version = "1.7.0", features = ["v4"] }
log = "0.4.20"
```

After setting the dependencies, let's install them using:

```bash
cargo run
```

Important items to check: 
* [Scylla](https://crates.io/crates/scylla): using the latest driver release.
* [Charybdis ORM](https://github.com/nodecosmos/charybdis): Charybdis is a ORM layer on top of scylla_rust_driver focused on easy of use and performance.
* [Uuid](https://crates.io/crates/uuid): help us to create UUIDs in our project
* [Actix](https://actix.rs/docs/): Rust Web Framework.
* [This Error](https://crates.io/crates/thiserror): Idiomatic Error Handling.
* [Chrono](https://crates.io/crates/chrono): DateTime/Timestamp Handling.

### 1.4 Project Structure

This is how it will be our source folder at the end of this tutorial:

```
/src
├── main.rs
├── config
│   ├── app.rs
│   ├── config.rs
│   └── mod.rs
├── http
│   ├── controllers
│   │   ├── leaderboard_controller.rs
│   │   ├── mod.rs
│   │   └── submissions_controller.rs
│   ├── mod.rs
│   └── requests
│       ├── leaderboard_request.rs
│       ├── mod.rs
│       └── submission_request.rs
└── models
    ├── leaderboard.rs
    ├── mod.rs
    └── submission.rs
```



## 2. Before Development

We're about to develop a feature per time. However we have to set some items like configuration and migrations before jump into the implementations!

```
/src
├── main.rs
└── config
    ├── app.rs
    ├── config.rs
    └── mod.rs
```

### 2.1 Setup the enviroment

Step by step of the configuration files needed on this project:

#### 2.1.1 Configuring the Environment

Here we going store some **environment variables** to use in in the project. Here's the structure: 

```rust
// File: src/config/config.rs

use serde::Serialize;

#[derive(Clone, Debug, Serialize)]
pub struct App {
    pub name: String,
    pub version: String,
    pub url: String,
    pub port: String,
}

#[derive(Clone, Debug, Serialize)]
pub struct Database {
    pub nodes: Vec<String>,
    pub username: String,
    pub password: String,
    pub cached_queries: usize,
    pub keyspace: String,
}

#[derive(Clone, Debug, Serialize)]
pub struct Config {
    pub app: App,
    pub database: Database
}

impl Config {
    pub fn new() -> Self {
        Config {
            app: App {
                name: dotenvy::var("APP_NAME").unwrap(),
                version: dotenvy::var("APP_VERSION").unwrap(),
                url: dotenvy::var("APP_URL").unwrap(),
                port: dotenvy::var("APP_PORT").unwrap(),
            },
            database: Database {
                nodes: dotenvy::var("SCYLLA_NODES").unwrap().split(',').map(|s| s.to_string()).collect(),
                username: dotenvy::var("SCYLLA_USERNAME").unwrap(),
                password: dotenvy::var("SCYLLA_PASSWORD").unwrap(),
                cached_queries: dotenvy::var("SCYLLA_CACHED_QUERIES").unwrap().parse::<usize>().unwrap(),
                keyspace: dotenvy::var("SCYLLA_KEYSPACE").unwrap()
            }
        }
    }
}


```

#### 2.1.2 Configuring the App State

The AppState is the main handler for Actix Web maintain the data during the Rust application Runtime. 

So, we'll be setting up the Database Connection for *Charybdis*, which expect a `CachingSession` to run queries under the ORM.

```rust
// file: src/config/app.rs

use std::sync::Arc;
use std::time::Duration;
use dotenvy::dotenv;
use scylla::{CachingSession, Session, SessionBuilder};
use crate::config::config::Config;

#[derive(Debug, Clone)]
pub struct AppState {
    pub config: Config,
    pub database: Arc<CachingSession>
}

impl AppState {
    pub async fn new() -> Self {
        dotenv().expect(".env file not found");

        let config = Config::new();
        let session: Session = SessionBuilder::new()
            .known_nodes(config.database.nodes)
            .connection_timeout(Duration::from_secs(5))
            .user(config.database.username, config.database.password)
            .build()
            .await
            .expect("Connection Refused. Check your credentials and IP linked on the ScyllaDB Cloud.");

        session.use_keyspace("leaderboard", false).await.expect("Keyspace not found");

        AppState {
            config: Config::new(),
            database: Arc::new(CachingSession::from(session, config.database.cached_queries))
        }
    }
}
```

#### 2.1.3 Making it Public

Under the `src/config/mod.rs` make sure to set the modules as public.

```rust 
// file: src/config/mod.rs

pub mod app;
pub mod config;
```


#### 2.1.4 Configuring the Runner

The last part of our configuration is the Web server to start our app. Here we need to call Actix HTTP and tell him what we're going to be passing under the AppState.

```rust 

use actix_web::{App, HttpServer};
use actix_web::web::Data;

use crate::config::app::AppState;

mod config;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let app_data = AppState::new().await;

    println!("Web Server Online!");
    println!("Listening on http://{}:{}", app_data.config.app.url, app_data.config.app.port);
    HttpServer::new(move || {
        App::new()
            .app_data(Data::new(AppState::new()))
    }).bind((
        app_data.config.app.url,
        app_data.config.app.port.parse::<u16>().unwrap()
    ))?.run().await
}
```

<div class="admonition note">
    <p class="admonition-title">Note</p>
    <p>
        At the <strong> bind() </strong>, you should add the Base URL and Port for your server to run. 
    </p>
</div>

Great! Now we're good to work on our Migrations!


### 2.2 Migrations and Models

Our goal is to make the development of this project in a good shape and easier to maintain. Thinking on that, I decided to use [Charybdis ScyllaDB ORM](https://github.com/nodecosmos/charybdis) because they have a really good migration tool for migration.

```
/src
├── main.rs
├── config
│   ├── app.rs
│   ├── config.rs
│   └── mod.rs
└─── models <- 
    ├── leaderboard.rs
    ├── mod.rs
    └── submission.rs
```

#### 2.2.1 Setting up the Models

At our project we'll be using two models, which will be: `submission.rs` and `leaderboard.rs`. At our design, we set the scope for each query which using Charybdis will be modeled like this:


```rust
// file: src/models/submssions.rs

use charybdis::macros::charybdis_model;
use charybdis::types::{Frozen, Int, Set, Text, Timestamp, Uuid};
use serde::{Deserialize, Serialize};


#[charybdis_model(
table_name = submissions,
partition_keys = [id],
clustering_keys = [played_at],
global_secondary_indexes = [],
local_secondary_indexes = [],
table_options = "
  CLUSTERING ORDER BY (played_at DESC)
",
)]
#[derive(Serialize, Deserialize, Default, Clone, Debug)]
pub struct Submission {
    pub id: Uuid,
    pub song_id: Text,
    pub player_id: Text,
    pub modifiers: Frozen<Set<Text>>,
    pub score: Int,
    pub difficulty: Text,
    pub instrument: Text,
    pub played_at: Timestamp,
}
```

```rust
// file: src/models/leaderboard.rs

use charybdis::macros::charybdis_model;
use charybdis::types::{Frozen, Int, Set, Text, Timestamp, Uuid};
use serde::{Deserialize, Serialize};

#[charybdis_model(
table_name = song_leaderboard,
partition_keys = [song_id, modifiers, difficulty, instrument],
clustering_keys = [player_id, score],
global_secondary_indexes = [],
local_secondary_indexes = [],
table_options = "
  CLUSTERING ORDER BY (score DESC, player_id ASC)
",
)]
#[derive(Serialize, Deserialize, Default, Clone, Debug)]
pub struct Leaderboard {
    pub id: Uuid,
    pub song_id: Text,
    pub player_id: Text,
    pub modifiers: Frozen<Set<Text>>,
    pub score: Int,
    pub difficulty: Text,
    pub instrument: Text,
    pub played_at: Timestamp,
}

```

Charybdis has the `charybdis_model` macro that allows you to create all the possible queries and create migrations from it.

To finish the setup, make sure to set as public the structs on `src/models/mod.rs`

```rust
// file: src/models/mod.rs

pub mod leaderboard;
pub mod submission;
```

#### 2.2.2 Migrating your Database. 

Now it's time! Get your credentials and migrate the your models to a ScyllaDB Cluster using the command below in the project root!

```shell
migrate -u scylla -p your-password --host your-node.clusters.scylla.cloud --keyspace leaderboard -d

# Detected 'src/models' directory

# Detected first migration for: submissions Table!
# Running CQL: CREATE TABLE IF NOT EXISTS submissions
# (
#    difficulty Text,
#    id Uuid,
#    instrument Text,
#    modifiers Frozen < Set < Text > >,
#    played_at Timestamp,
#    player_id Text,
#    score Int,
#    song_id Text,
#    PRIMARY KEY ((id) ,played_at)
# )
# WITH
#  CLUSTERING ORDER BY (played_at DESC)

# CQL executed successfully! ✅


# Detected first migration for: song_leaderboard Table!
# Running CQL: CREATE TABLE IF NOT EXISTS song_leaderboard
# (
#     difficulty Text,
#     id Uuid,
#     instrument Text,
#     modifiers Frozen < Set < Text > >,
#     played_at Timestamp,
#     player_id Text,
#     score Int,
#     song_id Text,
#     PRIMARY KEY ((song_id, modifiers, difficulty, instrument) ,player_id, score)
# )
#  WITH
#   CLUSTERING ORDER BY (player_id ASC, score DESC)

# CQL executed successfully! ✅
```

Now we're good to start developing the Web Features! 




## 3. Feature Development

With everything set-up, the next step is to develop each endpoint using Actix and Charybdis. 

```
/src
├── main.rs
├── config
│   ├── app.rs
│   ├── config.rs
│   └── mod.rs
├── http <- 
│   ├── controllers
│   │   ├── leaderboard_controller.rs
│   │   ├── mod.rs
│   │   └── submissions_controller.rs
│   ├── mod.rs
│   └── requests
│       ├── leaderboard_request.rs
│       ├── mod.rs
│       └── submission_request.rs
└─── models 
    ├── leaderboard.rs
    ├── mod.rs
    └── submission.rs
```


### 3.1 Feature: Submission Endpoint


#### 3.1.1 Submission: Data Transfer Object

To persist data into our ScyllaDB Database, we need to shape it with the right fields and types. So, so let's create a DTO to store it for us.


```rust
// file: src/http/requests/submission_request.rs

use charybdis::types::{Frozen, Int, Set, Text};
use serde::Deserialize;
use validator::Validate;

#[derive(Deserialize, Debug, Validate)]
pub struct SubmissionDTO {
    pub song_id: Text,
    pub player_id: Text,
    pub modifiers: Frozen<Set<Text>>,
    pub score: Int,
    pub difficulty: Text,
    pub instrument: Text,
}

```

At this DTO, we'll be adding the `Validate` and `Deserialize` derives since this fields will be received from a **JSON Payload**. 

Add it to the folder module:

```rust
//file: src/http/requests/mod.rs

pub mod submission_request;
```


#### 3.1.2 Submission: Model from Request

Let's keep our code well structured and create an Model from our DTO using a brand new function `from_request()`. 

```rust
// file: src/models/submission.rs

use charybdis::macros::charybdis_model;
use charybdis::types::{Frozen, Int, Set, Text, Timestamp, Uuid};
use serde::{Deserialize, Serialize};


use crate::http::requests::submission_request::SubmissionDTO;

#[charybdis_model(
table_name = submissions,
partition_keys = [id],
clustering_keys = [played_at],
global_secondary_indexes = [],
local_secondary_indexes = [],
table_options = "
  CLUSTERING ORDER BY (played_at DESC)
",
)]
#[derive(Serialize, Deserialize, Default, Clone, Debug)]
pub struct Submission {
    #[serde(default = "Uuid::new_v4")]
    pub id: Uuid,
    pub song_id: Text,
    pub player_id: Text,
    pub modifiers: Frozen<Set<Text>>,
    pub score: Int,
    pub difficulty: Text,
    pub instrument: Text,
    pub played_at: Timestamp,
}

impl Submission {
    pub fn from_request(payload: &SubmissionDTO) -> Self {
        Submission {
            id: Uuid::new_v4(),
            song_id: payload.song_id.to_string(),
            player_id: payload.player_id.to_string(),
            difficulty: payload.difficulty.to_string(),
            instrument: payload.instrument.to_string(),
            modifiers: payload.modifiers.to_owned(),
            score: payload.score.to_owned(),
            played_at: chrono::Utc::now(),
            ..Default::default()
        }
    }
}

```

#### 3.1.3 Submission: Controller

Actix allows you to parse a payload from JSON requests with a specific struct defined previously. So, we'll be using the `SubmissionDTO` as our field validation, build a model and insert it on the database. 

```rust
// file: src/http/controllers/submissions_controller.rs

use actix_web::{HttpResponse, post, Responder, Result, web};
use charybdis::operations::Insert;
use serde_json::json;
use validator::Validate;

use crate::config::app::AppState;
use crate::http::requests::submission_request::SubmissionDTO;
use crate::http::SomeError;
use crate::models::submission::Submission;

#[post("/submissions")]
async fn post_submission(
    data: web::Data<AppState>,
    payload: web::Json<SubmissionDTO>,
) -> Result<impl Responder, SomeError> {
    let validated = payload.validate();
    
    let response = match validated {
        Ok(_) => {
            let submission = Submission::from_request(&payload);
            submission.insert().execute(&data.database).await?;

            HttpResponse::Ok().json(json!(submission))
        }
        Err(err) => HttpResponse::BadRequest().json(json!(err)),
    };

    Ok(response)
}

```

Add it to the folder module:

```rust
//file: src/http/requests/mod.rs

pub mod submission_controller;
```


### 3.2 Feature: Leaderboard Ingestion


#### 3.1.1 Submission: Data Transfer Object

To persist data into our ScyllaDB Database, we need to shape it with the right fields and types. So, so let's create a DTO to store it for us.


```rust
// file: src/http/requests/submission_request.rs

use charybdis::types::{Frozen, Int, Set, Text};
use serde::Deserialize;
use validator::Validate;

#[derive(Deserialize, Debug, Validate)]
pub struct SubmissionDTO {
    pub song_id: Text,
    pub player_id: Text,
    pub modifiers: Frozen<Set<Text>>,
    pub score: Int,
    pub difficulty: Text,
    pub instrument: Text,
}

```

At this DTO, we'll be adding the `Validate` and `Deserialize` derives since this fields will be received from a **JSON Payload**. 

#### 3.1.2 Submission: Model from Request

Let's keep our code well structured and create an Model from our DTO using a brand new function `from_request()`. 

```rust
// file: src/models/submission.rs

use charybdis::macros::charybdis_model;
use charybdis::types::{Frozen, Int, Set, Text, Timestamp, Uuid};
use serde::{Deserialize, Serialize};


use crate::http::requests::submission_request::SubmissionDTO;

#[charybdis_model(
table_name = submissions,
partition_keys = [id],
clustering_keys = [played_at],
global_secondary_indexes = [],
local_secondary_indexes = [],
table_options = "
  CLUSTERING ORDER BY (played_at DESC)
",
)]
#[derive(Serialize, Deserialize, Default, Clone, Debug)]
pub struct Submission {
    #[serde(default = "Uuid::new_v4")]
    pub id: Uuid,
    pub song_id: Text,
    pub player_id: Text,
    pub modifiers: Frozen<Set<Text>>,
    pub score: Int,
    pub difficulty: Text,
    pub instrument: Text,
    pub played_at: Timestamp,
}

impl Submission {
    pub fn from_request(payload: &SubmissionDTO) -> Self {
        Submission {
            id: Uuid::new_v4(),
            song_id: payload.song_id.to_string(),
            player_id: payload.player_id.to_string(),
            difficulty: payload.difficulty.to_string(),
            instrument: payload.instrument.to_string(),
            modifiers: payload.modifiers.to_owned(),
            score: payload.score.to_owned(),
            played_at: chrono::Utc::now(),
            ..Default::default()
        }
    }
}

```

#### 3.1.3 Submission: Controller

Actix allows you to parse a payload from JSON requests with a specific struct defined previously. So, we'll be using the `SubmissionDTO` as our field validation, build a model and insert it on the database. 

```rust
// file: src/http/controllers/submissions_controller.rs

use actix_web::{HttpResponse, post, Responder, Result, web};
use charybdis::operations::Insert;
use serde_json::json;
use validator::Validate;

use crate::config::app::AppState;
use crate::http::requests::submission_request::SubmissionDTO;
use crate::http::SomeError;
use crate::models::submission::Submission;

#[post("/submissions")]
async fn post_submission(
    data: web::Data<AppState>,
    payload: web::Json<SubmissionDTO>,
) -> Result<impl Responder, SomeError> {
    let validated = payload.validate();
    
    let response = match validated {
        Ok(_) => {
            let submission = Submission::from_request(&payload);
            submission.insert().execute(&data.database).await?;

            HttpResponse::Ok().json(json!(submission))
        }
        Err(err) => HttpResponse::BadRequest().json(json!(err)),
    };

    Ok(response)
}

```


