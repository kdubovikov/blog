---
layout: post
title:  "Building yourself a weather station. Part 2"
date:   2020-06-21 00:00:00
categories: hardware
---

# Writing code for ESP32

In the [previous post](https://blog.kdubovikov.ml/articles/hardware/build-yourself-a-weather-station), we have covered the hardware setup for building a weather station. Starting from now, we will start coding. The first missing piece is the firmware for ESP32.

<!--more-->

Previous post in series: [Building a weather station]({% post_url 2020-06-13-build-yourself-a-weather-station %})

This firmware should read the data from BME280 and publish it to the MQTT topic. For ESP32 we have two options: using the Arduino framework or ESP-IDF. While Arduino is extremely popular and easy to use, I have decided to go with ESP-IDF for the following reasons:

- Although lower level, it gives you more control over the chip
- It packs a lot of functionality and is [well-documented](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/get-started/index.html). Look the features [here](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/index.html)
- It has a [set of examples](https://github.com/espressif/esp-idf/tree/master/examples) that demonstrate how you can work with the framework
- It has a configurable build system that allows you to create independent components that can be used by other developers.
- It supports [debugging](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/jtag-debugging/debugging-examples.html?highlight=debug) and [unit-testing](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/unit-tests.html?highlight=test)

Arduino hides a lot of complexity and requires less boilerplate code, but ESP-IDF is something that you will have more changes using in more serious projects so we will go with it to put ourselves in more realistic conditions.

# Code for this post

You can find the complete code at [github repository](https://github.com/kdubovikov/weather-station-esp).

# Setting up a project and connecting to ESP

ESP-IDF is written in C. If you are not familiar with the language, just go through [Learn C in Y minutes](https://learnxinyminutes.com/docs/c/), it will give you a good start.

After that, go through [getting started](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/get-started/index.html) guide at ESP-IDF documentation. It will cover how to download the framework and set up an example project. The most difficult part would be connecting to your ESP32 dev board via USB port. If you are using a Macbook with USB-C ports you will need to use a USB-C to USB adapter and a micro USB cable. I had no success with any kind of hub, but a simple adapter cable worked just fine.

{% maincolumn 'assets/img/esp32-weather-station/post-2/usb-c.jpg' %}

Be sure to install drivers, as advised on this [documentation page](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/get-started/establish-serial-connection.html). For Macbook pro you will need [CP210x drivers from Silicon Labs](https://www.silabs.com/products/development-tools/software/usb-to-uart-bridge-vcp-drivers). 

The reason you need drivers is that your computer does not understand the language ESP32 speaks. ESP32 transfers data to the external world using a serial protocol {% sidenote 1 "Serial protocol  means that data is transferred one bit at a time. The I2C protocol that we were discussing in the previous post is a serial protocol. On the opposite side are parallel protocols, which  use multiple wires to transmit several bits at once" %} named [RS-232](https://en.wikipedia.org/wiki/RS-232). To bridge serial interface with the USB interface your devboard uses a special bridge chip that acts as a protocol converter. In the case of my board, it is the CP210x. The problem is that your computer does not know how to handle data from your devboard. It assumes that it receives data encoded by some of the standard data transfer protocols of the USB world. Thunderbolt, for example. And in reality, it receives data encoded by a bridge chip. To solve this linguistic inconvenience you install a special driver that decodes signals from the bridge chip and masquerades itself as a virtual COM {% sidenote 2 "COM port is a computers serial inferface. Those ports were pretty common in desktop PCs but are long absent from modern laptops. Operating systems still have drivers for COM devices, so that's a natural choice for emulating serial communication" %} device.
 

# What will the firmware do?

We will write a firmware that will follow **the visious cycle of weather measurment**:

1. Connect to a WiFi network
2. Connect to MQTT server
3. Read data from BME280 sensor
4. Publish the newly read data to the MQTT topic
5. Go in deep sleep and wake after several hours to repeat all steps starting from 1

# Project structure

To continue, you may clone [this repository](https://github.com/kdubovikov/weather-station-esp) and explore the code along with the post. The code is separated into 4 main modules: 

- `weather_station.c` — the primary module that implements **the** **vicious cycle of weather measurement**
- `wifi.c` — everything related to WiFi connectivity
- `sensor.c`— the code for reading data from BME280 using I2C protocol
- `mqtt.c` — the code for connecting and publishing messages to MQTT server

All modules are stored in the [main](https://github.com/kdubovikov/weather-station-esp/tree/master/main) folder. We'll also need to mention our modules in the `CMakeLists.txt` so that ESP-IDF's build system would understand that we need all those source files to compile the project:

```
idf_component_register(SRCS "weather_station.c" "wifi.c" "sensor.c" "mqtt.c"
                       INCLUDE_DIRS ".")
```

Now, lets dive into each module and see how it works.

## Connecting to WiFi

To build this module, I have borrowed heavily from the [WiFi example](https://github.com/espressif/esp-idf/tree/release/v4.0/examples/wifi/getting_started/station) provided by the framework. In fact, the project was based on this template. The original example connects to the WiFi asynchronously, but a blocking call that waits until we got an IP address from the access point will be useful to us since we want to wait for a WiFi connection before going on and connecting to the MQTT server.

To create a blocking function we'll use a handy `EventGroup` functionality from the FreeRTOS{% sidenote 3 "Free Real Time Operating System contains lots of useful primitives for dealing with concurrency, multiprocessing and synchronization" %}. Events groups allow you to define an event which you can wait for in any other part of your code:

```c
// Define an event
EventGroupHandle_t* connected_event_group = (EventGroupHandle_t*) arg;

// ... code that connects to WiFi is ommited for brevity

// Hooray, we have connected. Fire the event
xEventGroupSetBits(*connected_event_group, WIFI_CONNECTED_BITS);

// ...
// You can wait for connection in some other function
xEventGroupWaitBits(*connected_event_group, WIFI_CONNECTED_BITS, true, true, portMAX_DELAY);
```

`xEventGroupWaitBits` is what `wifi_connect_blocking` function uses to block and wait while the board will establish a wifi connection. 

All WiFi-related code is located at the `wifi.c` file. The best function to start with is `wifi_connect_blocking`

```c
/**
 * @brief connect to WiFi network. Connection settings will be taken from sdkconfig file
 * 
 * @param connected_event_group will block on this event group until the connection is established
 */
void wifi_connect_blocking(EventGroupHandle_t* connected_event_group) {
    // initialize TCP adapter
    tcpip_adapter_init();

    // ESP_ERROR_CHECK is just a useful macro that checks that the function returned ESP_OK and logs error otherwise
    ESP_ERROR_CHECK(esp_event_loop_create_default());

    // initialize WiFi adapter and set default config
    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    // here we register our connected_event_group as a handler for all WiFi related events, 
    // as well as IP_EVENT that indicates that connection was established successfully
    ESP_ERROR_CHECK(esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID, &wifi_event_handler, connected_event_group));
    ESP_ERROR_CHECK(esp_event_handler_register(IP_EVENT, IP_EVENT_STA_GOT_IP, &wifi_event_handler, connected_event_group));

    // set up some settings from sdkconfig. You can use esp-idf.py menuconfig to change those variables interactively
    wifi_config_t wifi_config = {
        .sta = {
            .ssid = CONFIG_WIFI_SSID,
            .password = CONFIG_WIFI_PASSWORD,
            .scan_method = DEFAULT_SCAN_METHOD,
            .sort_method = DEFAULT_SORT_METHOD,
            .threshold.rssi = DEFAULT_RSSI,
            .threshold.authmode = DEFAULT_AUTHMODE,
        },
    };
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_config(ESP_IF_WIFI_STA, &wifi_config));

    // start up the WiFi module
    ESP_ERROR_CHECK(esp_wifi_start());

    // wait for connection
    xEventGroupWaitBits(*connected_event_group, WIFI_CONNECTED_BITS, true, true, portMAX_DELAY);
}
```

You can notice that we are registering WiFi event handlers that do the actual logic like this: `esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID, &wifi_event_handler, connected_event_group)`. The first two parameters scope our handler to catch any WiFi-related events with `wifi_event_handler` referenced by the third parameter. `connected_event_group` will be passed as an argument to the handler. Now, let's look at how the handler works:

```c
/**
 * @brief Initialize Wi-Fi as sta and set scan method 
 * 
 * @param arg a pointer to EventGroupHandle_t that will be used to send connected event
 * @param event_base set by ESP-IDF
 * @param event_id set by ESP-IDF
 * @param event_data set by ESP-IDF
 */
static void wifi_event_handler(void *arg, esp_event_base_t event_base,
                          int32_t event_id, void *event_data) {
    // cast our handler argument to EventGroupHandle_t
    EventGroupHandle_t* connected_event_group = (EventGroupHandle_t*) arg;

    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START)
    {
        // try to connect to WiFi nework if STAtion is started
        esp_wifi_connect();
    }
    else if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_DISCONNECTED)
    {
        // in case we were disconnected, try to connect again
        esp_wifi_connect();
    }
    else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP)
    {
        // log IP and fire our connected_event_group if the connection was successful
        ip_event_got_ip_t *event = (ip_event_got_ip_t *)event_data;
        ESP_LOGI(TAG, "got ip: %s", ip4addr_ntoa(&event->ip_info.ip));
        xEventGroupSetBits(*connected_event_group, WIFI_CONNECTED_BITS);
    }
}
```

The main purpose of the handler is to wait for the board to get a successful connection and fire `WIFI_CONNECTED_BITS` event to the event group. 

The next important part is to make readings from our BME280 sensor, so let's explore how to do that.

## Reading data from the sensor

Thanks to the great [esp-idf-lib](https://github.com/UncleRus/esp-idf-lib) component library there is only a single function related to the BME280. The library contains a ready to work component that interfaces with the sensor, so we don't need to write low-level I2C communication code ourselves. The creators of the library took If you want to study implemented all required procedures from the datasheet, including a compensation algorithm so that we have a simple interface to work with. If you want to study how this works, then refer to the B[MP280 component code](https://github.com/UncleRus/esp-idf-lib/blob/master/components/bmp280/bmp280.c) and the [datasheet](https://ae-bst.resource.bosch.com/media/_tech/media/datasheets/BST-BMP280-DS001.pdf).

The code related to sensor readings is located at `sensor.c` file:

```c
/**
 * @brief collect data from BME280 sensor
 * 
 * @param pvParamters a pointer to QueueHandle_t to which this function will send the data collected from the sensor
 */
void bmp280_collect_data(void *pvParamters)
{
    // cast parameter pointer to QueueHandle_t
    QueueHandle_t queue = (QueueHandle_t) pvParamters;

    // initialize the sensor
    bmp280_params_t params;
    bmp280_init_default_params(&params);
    bmp280_t dev;
    memset(&dev, 0, sizeof(bmp280_t));

    // we use pins 21 and 22 for data transfer between sensor and ESP32
    // note that BMP280_I2C_ADDRESS_0 holds a constant preset address 0x76
    // which is used by all BMP280 sensors
    ESP_ERROR_CHECK(bmp280_init_desc(&dev, BMP280_I2C_ADDRESS_0, 0, SDA_PIN, SCL_PIN));
    ESP_ERROR_CHECK(bmp280_init(&dev, &params));

    // check if we have detected BME280 or BMP280 modification of the sensor
    bool bme280p = dev.id == BME280_CHIP_ID;
    printf("BMP280: found %s\n", bme280p ? "BME280" : "BMP280");

    // the fun part starts here
    // read data from the sensor, create a WeatherMessage struct and send it to the queue
    float pressure, temperature, humidity;

    // we will wait for 500ms before reading any data
    vTaskDelay(500 / portTICK_PERIOD_MS);
    if (bmp280_read_float(&dev, &temperature, &pressure, &humidity) != ESP_OK)
    {
        ESP_LOGI(TAG, "Temperature/pressure reading failed\n");
    }
    else
    {
        // WeatherMessage is defined in weather_station.h
        struct WeatherMessage msg;
        msg.temperature = temperature;
        msg.humidity = humidity;
        msg.pressure = pressure;
        ESP_LOGI(TAG, "Sending WeatherMessage to queue: %f %f %f\n", msg.temperature, msg.pressure, msg.humidity);

        xQueueSendToFront(queue, (void *)&msg, xQueueBlockTime);
    }

    // this should be called if you want FreeRTOS to execute the task once and finish afterwards
    vTaskDelete(NULL);
}
```

This function collects data from the sensor, creates a `WeatherMessage` struct, and sends it to the queue which is later processed by the MQTT module, which we will look at next.

## Sending messages to MQTT server

`mqtt.c` module contains everything related to the MQTT. We will start by looking at the top-level function `mqtt_app_start`:

```c
/**
 * @brief starts the MQTT component
 * 
 * @param queue QueueHandle_t for WeatherMessages 
 */
void mqtt_app_start(QueueHandle_t queue)   
{
    ESP_LOGI(TAG, "mqtt_app_start QueueHandle: %p", queue);
    ESP_LOGI(TAG, "Connecting to MQTT server at %s", CONFIG_BROKER_URL);
    esp_mqtt_client_config_t mqtt_cfg = {
        .uri = CONFIG_BROKER_URL
    };

    esp_mqtt_client_handle_t client = esp_mqtt_client_init(&mqtt_cfg);
    // let's register our handler
    esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, (void*) queue);
    esp_mqtt_client_start(client);
}
```

Here, we initialize the MQTT client using a `CONFIG_BROKER_URL` which can be changed using ESP-IDF configuration system. After that, we register a handler for  MQTT events which will do all of the heavy lifting:

```c
/**
 * @brief A top-level handler that wraps around mqtt_event_handler_cb 
 * 
 * @param handler_args QueueHandle_t queue for WeatherMessages
 * @param base set by ESP-IDF
 * @param event_id set by ESP-IDF
 * @param event_data set by ESP-IDF
 */
static void mqtt_event_handler(void *handler_args, esp_event_base_t base, int32_t event_id, void *event_data)
{
    QueueHandle_t weather_msg_queue = (QueueHandle_t) handler_args;
    ESP_LOGI(TAG, "mqtt_event_handler QueueHandle: %p", weather_msg_queue);
    ESP_LOGD(TAG, "Event dispatched from event loop base=%s, event_id=%d", base, event_id);
    mqtt_event_handler_cb(event_data, weather_msg_queue);
}

/**
 * @brief MQTT event processing handler
 * 
 * @param event set by ESP-IDF
 * @param weather_msg_queue the queue to publish WeatherMessages
 * @return esp_err_t 
 */
static esp_err_t mqtt_event_handler_cb(esp_mqtt_event_handle_t event,  QueueHandle_t weather_msg_queue)
{
    // let's get MQTT client from our event handle
    esp_mqtt_client_handle_t client = event->client;
    switch (event->event_id)
    {
    case MQTT_EVENT_CONNECTED:
        // push WeatherMessage to the MQTT topic if we have connected to the server
        ESP_LOGI(TAG, "MQTT_EVENT_CONNECTED");
        send_weather_to_mqtt(&client, weather_msg_queue);
        break;
    case MQTT_EVENT_DISCONNECTED:
        // other events are handled mostly for logging and debugging,
        // the error messages should be self-explanatory
        ESP_LOGI(TAG, "MQTT_EVENT_DISCONNECTED");
        break;
    case MQTT_EVENT_ERROR:
        ESP_LOGE(TAG, "MQTT_EVENT_ERROR");
        if (event->error_handle->error_type == MQTT_ERROR_TYPE_ESP_TLS)
        {
            ESP_LOGE(TAG, "Last error code reported from esp-tls: 0x%x", event->error_handle->esp_tls_last_esp_err);
            ESP_LOGE(TAG, "Last tls stack error number: 0x%x", event->error_handle->esp_tls_stack_err);
        }
        else if (event->error_handle->error_type == MQTT_ERROR_TYPE_CONNECTION_REFUSED)
        {
            ESP_LOGE(TAG, "Connection refused error: 0x%x", event->error_handle->connect_return_code);
        }
        else
        {
            ESP_LOGW(TAG, "Unknown error type: 0x%x", event->error_handle->error_type);
        }
        break;
    default:
        ESP_LOGI(TAG, "MQTT recieved event id:%d", event->event_id);
        break;
    }
    return ESP_OK;
}
```

The main takeaway from this code is that when we get a `MQTT_EVENT_CONNECTED` event we trigger the `send_weather_to_mqtt` function which does exactly what it says:

```c
/**
 * @brief read message from the queue and send it to the MQTT topic
 * 
 * @param client MQTT client
 * @param weather_msg_queue FreeRTOS Queue which contains updates 
 */
static void send_weather_to_mqtt(esp_mqtt_client_handle_t *client, QueueHandle_t* weather_msg_queue)
{
    int msg_id;
    // if our queue handle points to somewhere
    if (weather_msg_queue != 0)
    {
        // wait for the new message to arrive
        ESP_LOGI(TAG, "Recieving message from the queue");
        struct WeatherMessage msg;
        if (xQueueReceive(weather_msg_queue, &(msg), pdMS_TO_TICKS(1000)))
        {
            ESP_LOGI(TAG, "Got new message from the queue");

            // encode WeatherMessage struct in JSON format using a helper function
            char json_msg[90];
            create_weather_msg(json_msg, &msg);

            // publish JSON message to the MQTT topic
            ESP_LOGI(TAG, "Sending JSON message: %s", json_msg);
            msg_id = esp_mqtt_client_publish(*client, "weather", json_msg, 0, 1, 0);
            ESP_LOGI(TAG, "sent publish successful, msg_id=%d", msg_id);

            // enable deep sleep mode for SLEEP_TIME ms
            // deep sleep lowers the power comsumption to about 0.15 mA, which will make out battery last much longer
            // TODO this is a bad place for this code since it clearly violates Single Responsibility Principle
            // This should be moved into separate function that will be called elsewhere
            const float SLEEP_TIME = 1.44 * 1e10;
            ESP_LOGI(TAG, "going to deep sleep for %.1f", SLEEP_TIME / 1e6);
            ESP_ERROR_CHECK(esp_wifi_stop());
            esp_deep_sleep(SLEEP_TIME);
        }
    }
}
```

It reads`WeatherMessage` structure from the queue and serializes it into a JSON message using `create_weather_msg`. Those are defined in a `weather_station.c` module, but let's look at them right now so you will have a complete picture of how the code works. The structure definition is really simple:

```c
struct WeatherMessage
{
    float temperature;
    float pressure;
    float humidity;
};
```

As well as the serialization code. It is not fancy and requires manual work each time you add a new field to the struct. However, it is simple, uses only the standard library, and works for our purposes well since the `WeatherMessage` struct won't change frequently, if at all.

```c
/**
 * @brief encode a WeatherMessage in JSON format
 * 
 * @param msg the output result will be written here
 * @param msg_struct WeatherMessage to be encoded
 */
void create_weather_msg(char *msg, struct WeatherMessage *msg_struct)
{
    sprintf(msg,
            "{\"temp\":%.2f,\"pressure\":%.2f,\"altitude\":%.2f,\"humidity\":%.2f}",
            msg_struct->temperature, msg_struct->pressure, 0.0, msg_struct->humidity);
}
```

Now, everything that's left is to explore how everything fits together in the `weather_station.c` module.

## Main function

The main function initializes all required resources and hands off the processing to the modules we have seen before:

```c
/**
 * @brief Main function of the allication. Orchestrates all tasks.
 * 1. Collects data from the BME280 sensor
 * 2. Starts the WiFi module and connects to the access point
 * 3. Sends collected data to the MQTT server
 * 4. Enters deep sleep and wakes after predetermined interval
 */
void app_main()
{
    // let's create an EventGroup which will be used to block until WiFi has connected
    EventGroupHandle_t s_connect_event_group = xEventGroupCreate();

    // a queue for WeatherMessages from the sensor
    QueueHandle_t queue = xQueueCreate(2, sizeof(struct WeatherMessage));

    ESP_LOGI(TAG, "QueueHandle: %p", &queue);

    esp_log_level_set("*", ESP_LOG_INFO);
    esp_log_level_set("esp-tls", ESP_LOG_VERBOSE);
    esp_log_level_set("MQTT_CLIENT", ESP_LOG_VERBOSE);
    esp_log_level_set("MQTT_EXAMPLE", ESP_LOG_VERBOSE);
    esp_log_level_set("TRANSPORT_TCP", ESP_LOG_VERBOSE);
    esp_log_level_set("TRANSPORT_SSL", ESP_LOG_VERBOSE);
    esp_log_level_set("TRANSPORT", ESP_LOG_VERBOSE);
    esp_log_level_set("OUTBOX", ESP_LOG_VERBOSE);

    // initialize I2C
    ESP_ERROR_CHECK(i2cdev_init());
    // this FreeRTOS function runs the bmp280_collect_data function in parallel
    // we don't need to block on it since we are using a queue to send WeatherMessages
    // so all functions will wait for new messages on the queue
    xTaskCreatePinnedToCore(bmp280_collect_data, "bmp280_collect_data", configMINIMAL_STACK_SIZE * 8, (void *)queue, 5, NULL, APP_CPU_NUM);

    // Initialize the NonVolatileStorage
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND)
    {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    // connect to the WiFi
    wifi_connect_blocking(&s_connect_event_group);

    // send all incoming WeatherMessages to the MQTT topic
    mqtt_app_start(queue);
}
```

Free Real Time Operating System contains lots of useful primitives for dealing with concurrency, multiprocessing and synchronization

# Setting up a Mosquitto server

The last remaining part is to set up an MQTT server, which will collect messages in a topic. Those messages could then be read by one or multiple subscribers. Of course, we could go the simples way and use the hosted solution, but we will set up our own server to see how everything works internally.

You can set up one on your machine {% sidenote 4 "Google Compute Cloud offers and [always-free f1-micro server](https://cloud.google.com/free/) that will be more than enough for our purposes. You can register a free domain name for it at [http://freenom.com](http://freenom.com/)." %}, or use a virtual private server on some cloud provider like Google Compute Cloud, AWS, Azure, Linode or DigitalOcean.

We will use a popular and reliable [Mosquitto](http://mosquitto.org) as an MQTT server. It is pretty simple to set up. At first, [download the software](https://mosquitto.org/download/) or install it using a package manager. For Ubuntu, you can simply install it via:

```bash
snap install mosquitto
```

After that, [create a user](https://mosquitto.org/man/mosquitto_passwd-1.html) so we would be able to set up a password-protected server:

```bash
mosquitto_passwd -c /etc/mosquitto/passwd mqttuser [password]
```

Next, we will need to configure the server. On Ubuntu, the configuration file could be found at `/etc/mosquitto/conf.d/default.conf`. Open it using your favourite editor and set up the password authentication:

```
# disable anonymous login
allow_anonymous false

# enable password file based authentication
password_file /etc/mosquitto/passwd

# use this if you want to use unencrypted HTTP connections
#listener 1883

# use this to connect without password from localhost
listener 1883 localhost

# use this if you want to use encrypted HTTPS connections
listener 8883
certfile /etc/letsencrypt/live/my-site.com/cert.pem
cafile /etc/letsencrypt/live/my-site.com/chain.pem
keyfile /etc/letsencrypt/live/my-site.com/privkey.pem
```

In case you have a public domain name I strongly suggest setting up SSL encryption using [LetsEncrypt](https://letsencrypt.org). Otherwise, the passwords will be sent via an unencrypted channel that is free to sniff by anyone.

That's it, you can now start the broker:

```bash
│sudo /usr/sbin/mosquitto -c /etc/mosquitto/mosquitto.conf -v
```

You can now  subscribe to the topic using a command-line client and start-up your ESP32 board:

```
mosquitto_sub -p 1883 -t weather 

# the -t argument sets the topic name
```

After a few moments, you should see a message from your board. Congratulations 🎉.

# Conclusion

In this post, we have seen how to create firmware for the ESP32 weather station and set up a Mosquitto MQTT server that will collect messages from the board.

You can set up multiple devices and point them at different topics if you want to collect weather from multiple locations ⛅️.

In the next post, we will build a backed that collects and stores weather data, a Telegram notification bot, and a REST API. A lot of new things to cover! 

Subscribe to the RSS feed to get notified when the next post comes out and share this one if you liked it.

Next post in the series: [Async Unicorns love Rust]({% post_url 2020-07-27-async-unicorns-love-rust %})