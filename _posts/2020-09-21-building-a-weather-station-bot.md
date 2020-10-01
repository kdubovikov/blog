---
layout: post
title:  "Building a Weather Station Bot"
date:   2020-09-19 00:00:00
categories: rust
---


In this post we are going to switch from hardware to software and write a telegram bot that will be the primary user interface for our weather station.
<!--more-->

# Writing a Weather Station Bot
In the [previous post](https://blog.kdubovikov.ml/articles/rust/async-unicorns-love-rust) we have covered the basics of the asynchronous programming in Rust, which will allow us to write a telegram bot that reads measurements from MQTT server and send notification messages to Telegram. The complete source code for this post is hosted in a GitHub repository.

This project will use a few libraries which will greatly simplify our work:

- [Diesel ORM](http://diesel.rs) Framework that will abstract away the details of communicating with the relational database in which we will store sensor readings history
- [tbot](https://tbot.rs) framework which wraps telegram's APIs into an asynchronous bot creation framework

## Bot architecture

Our bot will store all data received from ESP Weather Station in a relational database. In our case, SQLite will provide everything we need without complicating the setup. SQLite is stored in a single file and does not require installing any separate database management software while providing a fully-fledged SQL interface to our data.

We will also use a tbot framework for writing a bot. Tbot is based on tokio, that's why our asynchronous programming digression will become handy.

We will also use a library called [rumq-client](https://docs.rs/rumq-client/0.1.0-alpha.10/rumq_client/index.html) for communicating with our Mosquitto MQTT server.

We will package our bot as a command-line application which supports file-based configuration. Packages like clap and settings will help us to quickly implement config files and parsing command line arguments.

## Designing a database

First, let's look at `[schema.rs](http://schema.rs)` file which contain's our database table definitions. Based on this metadata diesel will be able to generate SQL migrations that create the tables in our DB, so we do not need to do this by hand. Those definitions also will be used by diesel to automatically create queries to the DB.

```rust
table! {
    // subscribers table will store telegram identifiers of all users 
    // who want to recieve notifications from our bot
    subscribers (id) {
        id -> Integer,
        telegram_chat_id -> BigInt,
    }
}

table! {
    // weather_log is an append-only log table that
    // will contain all sensor readings
    weather_log (id) {
        id -> Integer,
        timestamp -> Text, // date and time of the reading. SQLite does not have a dedicated type for storing dates, so we will use Text 
        temp -> Float,
        pressure -> Float,
        humidity -> Float,
    }
}

allow_tables_to_appear_in_same_query!(
    subscribers,
    weather_log,
);
```

The first thing we need to do before working with data is to connect to the database. Let's look at how we can do this with diesel:

```rust
// we will need those all imports for the db-related code
#[macro_use]
extern crate diesel;
mod schema;

use crate::schema::weather_log;
use crate::schema::subscribers;
use crate::schema::weather_log::dsl::*;
use crate::schema::subscribers::dsl::*;
use chrono::Utc;
use diesel::prelude::*;
use diesel::sqlite::SqliteConnection;
use serde::{Deserialize, Serialize};
use std::fmt::Display;

/// Connect to SQLite database
pub fn establish_connection(database_url: &str) -> SqliteConnection {
    println!("Connecting to {}", database_url);
    SqliteConnection::establish(database_url)
        .expect(&format!("Error connecting to {}", database_url))
}
```

As you can see, the connection code is pretty straightforward. 

### Subscribtions

Let's now decide how we will represent subscribers in the code:

```rust
/// This struct represents an existing bot subscriber who can recieve 
/// notifications about new weather measurements.
/// `Queryable` trait identifies that Diesel will use this structure to 
/// represent rows selected from  SQLite database.
#[derive(Queryable)]
pub struct Subscriber {
    id: i32,
    telegram_chat_id: i64
}

/// This struct represents a new bot subscriber.
/// `Insertable` trait means that Diesel will expect this stcuture to
/// represent new rows that we can insert into SQLite database.
#[derive(Insertable)]
#[table_name = "subscribers"]
pub struct NewSubscriber {
    telegram_chat_id: i64
}
```

Please note, that we can use `table_name` attribute to deliberately tell diesel which table to use as a backing storage for a struct.

Next, we need to define some basic functions to work with subscriptions:

```rust
// Save new subscriber to the database if he does not already exist
pub fn subscribe(chat_id: i64, connection: &SqliteConnection) -> Result<NewSubscriber, &str> {
    // let's find if there are any existing subscribers
    let existing_subscriber = subscribers.filter(telegram_chat_id.eq(chat_id)).first::<Subscriber>(connection);

    if let Err(diesel::NotFound) = existing_subscriber {
        // let's create a new one if there is noone found
        let subscriber = NewSubscriber {telegram_chat_id: chat_id };

        match diesel::insert_into(subscribers).values(&subscriber).execute(connection) {
            Ok(_) => Ok(subscriber),
            Err(_) => Err("Error while saving new subscriber to DB")
        }
    } else {
        Err("The subscriber already exists")
    }
}

/// Deletes a subscriber from the database
pub fn unsubscribe(chat_id: i64, connection: &SqliteConnection) -> QueryResult<usize> {
    diesel::delete(subscribers.filter(telegram_chat_id.eq(chat_id))).execute(connection)
}
```

### Weather messages

Now, it is time to define structures related to weather messages. It is important to mention that the design process of this kind of API is in general done in backwards. Instead of writing the DB code as a first step, it is more natural to write the bot first, deciding on which functionality you will need from the DB layer in ad-hoc manner. However, it is a lot easier to explain how everything works starting from the data model, so we will go through that first.

We will separate MQTT message format from the internal one to keep our backend decoupled from edge-device message format:

```rust
/// This structure represents weather data that we read from SQLlite
#[derive(Queryable, Serialize, Deserialize, Debug)]
pub struct WeatherMessage {
    pub id: i32,
    pub timestamp: String,
    pub temp: f32,
    pub pressure: f32,
    pub humidity: f32,
}

/// WeatherMessage respresentation for insert DB queries
/// This structure represents weather data that we insert into SQLlite
#[derive(Insertable, Serialize, Deserialize, Debug)]
#[table_name = "weather_log"]
pub struct NewWeatherMessage {
    timestamp: String,
    temp: f32,
    pressure: f32,
    humidity: f32,
}

/// Raw WeatherMessage that comves from the edge device via MQTT in a JSON format
#[derive(Serialize, Deserialize, Debug)]
pub struct EspWeatherMessage {
    temp: f32,
    pressure: f32,
    humidity: f32,
}
```

We will also extend `EspWeatherMessage` with a few formatting methods. We will use `Display` trait to be able to convert the message into a human-readable format:

```rust
impl EspWeatherMessage {
    pub fn temp_to_emoji(&self) -> &str {
        if self.temp < -10. {
            "🥶"
        } else if self.temp < 0. {
            "❄️"
        } else if self.temp > 20. {
            "☀️"
        } else if self.temp > 30. {
            "🔥"
        } else {
            ""
        }
    }

    /// A handy function to convert humidity percentage into useful notifications
    pub fn humidity_to_emoji(&self) -> &str {
        if self.humidity > 70. {
            "Do not forget an umbrella ☂️"
        } else if self.humidity > 90. {
            "🌧"
        } else {
            ""
        }
    }

    /// Converts pressure from Pascals to millimetres of mercury
    pub fn pressure_to_enoji(&self) -> &str {
        const PA_TO_MM_MERCURY: f32 = 133.322;
        const NORMAL_PRESSURE: f32 = 101_325.0 / PA_TO_MM_MERCURY;

        if (self.pressure / PA_TO_MM_MERCURY) > (NORMAL_PRESSURE + 10.0) {
            "⬆️ high pressure"
        } else if (self.pressure / PA_TO_MM_MERCURY) < (NORMAL_PRESSURE - 10.0) {
            "⬇️ low pressure"
        } else {
            ""
        }
    }
}

/// Display trait is used to convert `EspWeatherMessage` to string.
/// We will use this implementation to convert messages to text notifications that will be sent to subscribers
impl Display for EspWeatherMessage {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        const pa_to_mm_mercury: f32 = 133.322;

        write!(
            f,
            "{}{}{}\n℃{:>10.2}\nHumidity{:>10.2}%\nPressure{:>10.2}",
            self.temp_to_emoji(),
            self.humidity_to_emoji(),
            self.pressure_to_enoji(),
            self.temp,
            self.humidity,
            self.pressure / pa_to_mm_mercury
        )
    }
}
```

A few convenience methods for constructing and saving weather messages will also be handy to use in our bot:

```rust

impl NewWeatherMessage {
    /// Create a new WeatherMessage that later can be saved to database
    pub fn new(tmp: f32, press: f32, hum: f32) -> NewWeatherMessage {
        NewWeatherMessage {
            timestamp: Utc::now().to_rfc3339(),
            temp: tmp,
            pressure: press,
            humidity: hum,
        }
    }

    /// Converts `EspWeatherMessage` to `NewWeatherMessage` that can be saved to DB.
    /// This function should be used to convert all new incoming weather messages
    pub fn from_esp_weather_message(msg: &EspWeatherMessage) -> NewWeatherMessage {
        NewWeatherMessage::new(msg.temp, msg.humidity, msg.humidity)
    }
   
    /// Saves message to database
    pub fn save_to_db(&self, connection: &SqliteConnection) -> QueryResult<usize> {
        let result = diesel::insert_into(weather_log::table)
            .values(self)
            .execute(connection);
        
        result
    }
}
```

If you need more info abut diesel, I suggest going through their [Getting Started guide](http://diesel.rs/guides/getting-started/).

## Writing a bot

Now, let's go on to the most interesting part: writing a telegram bot. We will go through a fairly long listing of bot's `main` function, so I'll break it in pieces for better clarity. First, we will setup our command line arguments parsing using `clap` crate:

```rust
// Main WeatherStation Telegram bot fucntion
#[tokio::main]
async fn main() {
    // Clap library allows us to declaratively construct Command-Line Interfaces.
    // Here we use it to display application medatada and allow user to specify config path
    let matches = App::new("Weather station bot")
        .version("0.1.0")
        .author("Kirill Dubovikov <dubovikov.kirill@gmail.com>")
        .about("Telegram bot for ESP32 weather station")
        .arg(
            Arg::with_name("config")
                .short("c")
                .long("config")
                // Notice that we can set FILE types for arguments, so that clap will validate user input for us.
                // If the user will enter invalid path clap will notify him and end the application
                .value_name("FILE") 
                .help("Sets a custom config file")
                .required(true),
        )
        .get_matches();

    println!("⚠️Do not forget to make sure that you can connect to Telegram APIs. The polling module won't time out if the service is unawailable");
    
    // Read settings from the config file. "config" crate makes this simple
    let config = matches.value_of("config").unwrap_or("config");
    let settings = Settings::new(config).expect("Error while reading settings");
```

`Settings` above is just a serializable structure that holds our bot settings inside. Next, we will focus on setting up the `tbot` framework, along with several `tokio` channels that we will use to communicate between our async functions:

```rust
    // Structure that represents our Telegram bot.
    // It is wrapped in an Arc (Atomic reference counter) because we will use it later in send_message_to_telegram function.
    // This function is asynchronous, so Tokio could run it in a different thread.
    // Rust compiler is very smart and it won't allow us to pass values between different threads
    // without proper tracking of references and synchronization, which Arc provices for us.
    let bot = Arc::new(tbot::Bot::new(settings.telegram.token.clone()));

    // Tokio unbounded_channel is used to communicate between different asynchronous functions which may run in different threads.
    // Channels are like pipes: tok_tx can be used to send messages down the piple, and tok_rx can be used to recieve them
    let (tok_tx, mut tok_rx) = tokio::sync::mpsc::unbounded_channel::<EspWeatherMessage>();

    // watch::channel is a Tokio channel with a single producer and multiple consumers.
    // This is useful to share configuration (single producer) with many asynchronous functions (multiple consumers)
    let (_, conf_rx) = watch::channel(settings.clone());

    let mut conf = conf_rx.clone();
    
    // That's how we can recieve a config from watch::channel
    let settings = conf.recv().await.unwrap();
```

Then, let's spawn a function that will read incoming messages from our MQTT queue:

```rust
    // tokio::spawn runs process_mqtt_messages asynchronously so that it won't block our main thread.
    // This means that the execution of this function can continue without interruption.
    //
    // async move designates that we can use variables from our main thread inside tokio::spawn block
    // If you are familiar with closures and Rust borrow checker: 
    // async move tells that we can move values from enclosing environment inside the clousure's environment
    tokio::spawn(async move {
        process_mqtt_messages(&settings, tok_tx).await;
    });
```

Here it the MQTT related code that does all of the actual job:

```rust
/// Connects to MQTT server using [Settings](settings::Settings) structure. The settings are meant to be read from config TOML file
/// Will automatically subsribe to the topic name in the config.
/// Subscribes to the weather topic and forwards parsed messages to tokio channel
async fn process_mqtt_messages(settings: &Settings, tx: UnboundedSender<EspWeatherMessage>) {
    println!(
        "Conntcting to MQTT server at {}:{}/{}",
        settings.mqtt.host, settings.mqtt.port, settings.mqtt.topic_name
    );

    // Create MQTT connection options using information from config file
    let mut mqtt_options = MqttOptions::new("weather_station_bot", settings.mqtt.host.clone(), settings.mqtt.port.clone());
    mqtt_options.set_credentials(settings.mqtt.username.clone(), settings.mqtt.password.clone());
    mqtt_options.set_inflight(10);

    let ca_cert = read_file_to_bytes(&settings.tls.ca_cert);
    mqtt_options.set_ca(ca_cert);
    mqtt_options.set_keep_alive(50);
    mqtt_options.set_throttle(std::time::Duration::from_secs(1));

    // requests_tx will be used to send subscription requests to the MQTT server
    // requests_rx will be used by tokio event loop to recieve new messages
    let (mut requests_tx, requests_rx) = channel(10);

    // Here we subscribe to the MQTT topic from the config file
    let subscription = Subscribe::new(settings.mqtt.topic_name.clone(), QoS::AtLeastOnce);
    let _ = requests_tx.send(Request::Subscribe(subscription)).await;

    // And create the Tokio event loop which drives the whole message processing
    let mut event_loop = eventloop(mqtt_options, requests_rx);
    let mut stream = event_loop.connect().await.unwrap();

    // At last, we delegate each new message process_message_from_device function
    println!("Waiting for notifications");
    while let Some(notification) = stream.next().await {
        println!("New notification — {:?}", notification);
        process_message_from_device(&notification, &tx);
    }
}

/// Main MQTT message processing loop. 
///
/// Recieves a message from MQTT topic, deserializes it and sends it for further processing using Tokio MPSC framwrok. See [send_message_to_telegram](send_message_to_telegram)
fn process_message_from_device(notification: &Notification, tok_tx: &UnboundedSender<EspWeatherMessage>) {
    match notification {
        // Notification::Publish represents a message published in MQTT topic
        Notification::Publish(publish) => {
            let text: String = String::from_utf8(publish.payload.clone())
                .expect("Can't decode payload for notification");
            println!("Recieved message: {}", text);

            // As you remember, our ESP32 board encodes messages in JSON format and sends then to the MQTT server.
            // Here, we decode (deserialize) this message into Rust struct `EspWeatherMessage`
            let msg: EspWeatherMessage = serde_json::from_str(&text)
                .expect("Error while deserializing message from ESP");
            println!("Deserialized message: {:?}", msg);
            println!("{}", msg);

            // We send deserialized message via Tokio channel, that allows different coroutines to communicate between each other
            tok_tx.send(msg).unwrap();
        }
        _ => println!("{:?}", notification),
    }
}
```

Now that we have handled the MQTT message processing part, let's use the `tok_rx` channel to read messages and send them over to our subscribers:

```rust
    // We clone some variables since we need to move them into closure, but we allso will need them later
    // Alternatively, you can use Arc's or channels to curcumvent cloning, but I have decided to
    // make things simpler since cloning values a constant number of times at the application start
    // won't be a bottleneck in our case
    let bot_sender = bot.clone();
    let mut conf = conf_rx.clone();
    tokio::spawn(async move {
        // Here all the magic happens 🌈
        let settings: Settings = conf.recv().await.unwrap();
        // Recieve new message from MQTT topic
        while let Some(msg) = tok_rx.recv().await {
            // Get all subrcribers from database
            let subscribers = get_all_subscribers(&establish_connection(&settings.db_path)); 
            println!("Recieved new message — {:?}", msg);
            let db_path = settings.db_path.clone();

            // Send message to all active subscribers
            for subscriber in &subscribers {
                send_message_to_telegram(*subscriber, &msg, &bot_sender).await;
            }

            // Save weather data to database. Here we use a spawn blocking function to execute blocking code
            // which won't normally work in an async block
            tokio::task::spawn_blocking(move || {
                println!("Saving message to DB");
                let connection = establish_connection(&db_path); 
                // Convert ESPWeatherMessage to NewWeatherMessage which can be used by diesel framework
                // to save weather data to database
                let new_log = NewWeatherMessage::from_esp_weather_message(&msg);
                new_log.save_to_db(&connection).unwrap();
                print!("Successfully saved message to DB");
            });
        }
    });
```

Here is the `send_message_to_telegram` function that we used in the code above:

```rust
/// Sends a message to subscribers
async fn send_message_to_telegram(chat_id:i64, msg: &EspWeatherMessage, bot: &Arc<tbot::Bot>) {
    // First, we convert EspWeatherMessage to string. Since we have implemented Diplay trait, we can just use format! macro
    let message_str = &format!("{}", msg);
    // Text::plain is used in tbot Telegram library to wrap plain text messages
    let message = Text::plain(message_str);
    println!("Sending message to Telegram");

    // Here, we send the message to a subscriber's chat
    bot.send_message(ChatId::from(chat_id), message)
        .call()
        .await // send_message is asynchronous, to actually call it and wait for it's result we need to use await
        .expect("Error while sending message to the bot");
}
```

Now that we are done with sending messages via Telegram, we need allow our users to subscribe to our notifications. For this, we will implement `/subscribe` and `/unsubscribe` commands:

```rust
// Here we get the event loop to register some commands for the bot users
let mut event_loop = (*bot).clone().event_loop();

let conf = conf_rx.clone();
    event_loop.command("subscribe", move |context| {
        // Subscribe command can be used by the user to get notifications about
        // new weather readings
        let mut conf = conf.clone();

        async move {
            let settings: Settings = conf.recv().await.unwrap();
            // Here we get the user's chat_id
            let chat_id = context.chat.id.0;
            context
                .send_message(&format!("Your chat id is {}", chat_id))
                .call()
                .await
                .err();
            

            // and save it to the database
            let connection = establish_connection(&settings.db_path); 
            subscribe(chat_id, &connection).unwrap();
        }
    });

    let conf = conf_rx.clone();
    event_loop.command("unsubscribe", move |context| {
        // the unsubscribe command can be used to unsubscribe from all bot notifications
        let mut conf = conf.clone();
        async move {
            let settings: Settings = conf.recv().await.unwrap();
            let chat_id = context.chat.id.0;
            let connection = establish_connection(&settings.db_path);
            let result = unsubscribe(chat_id, &connection);

            if result.is_ok() {
                context.send_message("Sucessfully unsubscribed").call().await.err();
            } else {
                context.send_message("Can't unsubscribe. Are you subscribed?").call().await.err();
            }
        }
    });
```

That's it. We have implemented a basic notification bot. Now just need to start the event loop and the bot will process messages from our weather station and send them over to the subscribers:

```rust
// this starts the main event loop
event_loop.polling().start().await.unwrap();
```

You may now compile the project, fill in the config file and start the bot like this:

```bash
./target/debug/weather_station_bot --config config.toml
```

# Conclusion

We have created a Telegram bot that is capable of managing user subscriptions and sending weather notifications from our weather station. In the next part, we will create an alternative user interface: a web dashboard along with a REST API for our weather database.