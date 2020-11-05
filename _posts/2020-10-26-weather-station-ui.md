---
layout: post
title:  "Building a Weather Station UI"
date:   2020-11-05 00:00:00
categories: rust ui
---

In this last post of the series we are going to look at how to build a simple weather station UI dashboard.

<!--more-->

# Table of Contents

1. [Build yourself a weather station. Part I](/articles/hardware/build-yourself-a-weather-station)
1. [Building yourself a weather station. Part 2](/articles/hardware/build-yourself-a-weathe-station-part-2)
1. [Async Unicorns love Rust](/articles/rust/async-unicorns-love-rust)
1. [Building a Weather Station Bot](/articles/rust/building-a-weather-station-bot)
1. → Building a Weather Station UI

# Let's build a user interface
In [the previous post](/articles/rust/building-a-weather-station-bot) we have built a weather station bot that can notify us about new measurements made by our weather station. Today, we are going to build a simple REST API to fetch the data from our server and draw a chart on a simple dashboard.

# REST API
Our web interface will need historical data that will be displayed in the charts. Before building an actual UI we will create a REST API that will return all sensor readings that are stored in our database. The code for the API is located in the [`weather_station_api.rs`](https://github.com/kdubovikov/weather-station-bot/blob/master/src/bin/weather_station_api.rs) file. We will use the [tower_web](https://docs.rs/tower-web/0.3.7/tower_web/) crate to create a service that will return all sensor measurements along with their timestamps. We already have the database related code covered in the previous post. The entire service code is very concise:

```rust
/// This type will be part of the web service as a resource.
#[derive(Clone, Debug)]
struct WeatherApi;

/// This will be the JSON response
#[derive(Response)]
struct WeatherMessageResponse {
    messages: Vec<WeatherMessage>
}

impl_web! {
    impl WeatherApi {
        #[get("/")]
        #[content_type("json")]
        fn get_all_weather_messages(&self) -> Result<WeatherMessageResponse, ()> {
            let conn = establish_connection("./db.sqlite"); // this is a magic string better to be left in a config file, but I'll let it as it is for the sake of simplicity
            let weather_messages = get_all_weather_messages(&conn);
            Ok(
               WeatherMessageResponse { messages: weather_messages }
            )
        }
    }
}
```

After that all we need is to run the server somewhere inside our `main` function:

```rust
let cors = CorsBuilder::new()
        .allow_origins(AllowedOrigins::Any { allow_null: true } )
        .build();

ServiceBuilder::new()
    .resource(WeatherApi)
    .middleware(cors)
    .run(&addr)
    .unwrap();
```

You can check that the API works as intended by running the server and issuing an HTTP GET request with `curl` or [Postman](https://www.postman.com).

# Creating a UI Dashboard
Now, let's create a simple UI Dashboard. The purpose of our interface will be to display a historical plot of temperature, humidity and pressure readings from our weather station. We won't use any complex javascript frameworks and stick to vanilla javascript. This is how the final result will look like:

{% fullwidth assets/img/esp32-weather-station/post-1/weather-dashboard.gif %}

## Markup
For a start, let's look at `index.html`:

```html
<html>
<head>
	<link rel="stylesheet" href="styles/weather_station.css" />
	
    <!-- load fonts -->
	<link href="https://fonts.googleapis.com/css2?family=Open+Sans:wght@300&display=swap" rel="stylesheet">
    
    <!-- a few libraries that we will use -->
    <!-- a css framework for animations -->
	<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/animate.css/4.0.0/animate.min.css" />

    <!-- chart.js for plots -->
	<script src="https://cdn.jsdelivr.net/npm/chart.js@2.9.3/dist/Chart.min.js"></script>

    <!-- moment.js for working with dates -->
	<script src="https://cdnjs.cloudflare.com/ajax/libs/moment.js/2.26.0/moment.min.js"></script>

    <!-- our main script, we will look into this later 😉 -->
	<script src="index.js"></script>
</head>
<body>
    <!-- let's use HTML5 section tags to group -->
    <!-- related content together instead of using ambiguous div's -->
	<header>
		<div class="weatherIcon">
			<object data="img/weather_icon.svg" type="image/svg+xml"></object>
		</div>
		
		<h1>How's the <span class="accent">weather</span>?</h1>
	</header>
	
	<section>
        <!-- this is a group of radio buttons --> s
        <!-- that user can click to switch between different metrics -->
		<div class="group">
			<input type="radio" name="rb" id="temp_radio" />
		    <label for="temp_radio">Temperature</label>
		    <input type="radio" name="rb" id="humidity_radio" />
		    <label for="humidity_radio">Humidity</label>
		    <input type="radio" name="rb" id="pressure_radio" />
		    <label for="pressure_radio">Pressure</label>
		</div>
		
        <!-- a canvas that we will pass on to chart.js in our main script -->
		<div class="chartContainer">
			<canvas id="tempChart" width="50" height="50"></canvas>
		</div>
	</section>
</body>
</html>
```

## UI logic
Having finished with the markup we now can transition to making our dashboard to be useful by implementing some logic in the `index.js` script. As a headstart, let's start from the top abstraction level and look what the code does in overall:

```javascript
const API_URL = "http://localhost:8080";

// this will be called by browser as soon as
// all necessary resources were loaded and the page
// is ready to render
window.onload = async () => {
    // fetch data from our API
    // we use asynchronous calls to block execution only when necessary
    const data = await (await fetch(API_URL)).json()

    // convert timestamps to the label format we want to use in our charts
    const labels = data.messages.map(item => moment(item.timestamp).format('ddd HH a'));

    // render chart
    let chart = createChart(labels, data);

    // create handlers for radio buttons that can be used to select different metrics
    // that will be displayed on the chart
    document.getElementById("temp_radio").onclick = () => {
        // this call will prepare data to be used in a 
        // format that chart.js expects as an input
        const tempData = createChartDataset(data, 'temp');

        // this function will update the initialized chart with
        // new data, along with its minimal and maximal bounds
        updateChart(chart, tempData, tempData.min - 5, tempData.max + 5);
    };

    document.getElementById("pressure_radio").onclick = () => {
        const pressureData = createChartDataset(data, 'pressure');
        updateChart(chart, pressureData, 700, pressureData.max + 10);
    };

    document.getElementById("humidity_radio").onclick = () => {
        const humidityData = createChartDataset(data, 'humidity');
        updateChart(chart, humidityData, 0, 100);
    };

    // set default metric to temperature
    document.getElementById('temp_radio').click();
};
```

Now that we have seen the whole picture, let's look at how the individual functions work. We will start with `createChart`, that uses our canvas to display a plot using chart.js library:

```javascript
/**
 * Creates a new chart
 * @param {Array.<string>} labels 
 * @param {Array} data 
 */
function createChart(labels, data) {
    let ctx = document.getElementById('tempChart').getContext('2d');
    // Set colors and fonts
    Chart.defaults.global.defaultFontColor = '#636160ff';
    Chart.defaults.global.defaultFontFamily = '"Open Sans", sans-serif';
    Chart.defaults.global.defaultFontSize = 20;

    // Create chart 
    let chart = new Chart(ctx, {
        "type": "line",
        "data": {
            "labels": labels,
            "datasets": [{
                "label": "",
                "data": data,
                "fill": false,
                "borderColor": "#6e2594ff",
                "lineTension": 0.1
            }]
        },
        "options": {
            "legend": {
                "display": false
            },
            "aspectRatio": 1,
            "maintainAspectRatio": false,
            "scales": {
                "yAxes": [{
                    "offset": true,
                    "gridLines": {
                        "display": false
                    },
                    "ticks": {
                        "suggestedMin": 0,
                        "suggestedMax": 35
                    }
                }],
                "xAxes": [{
                    "offset": true,
                    "gridLines": {
                        "display": false
                    }
                }]
            }
        }
    });

    return chart;
}
```

Now, let's look at how we can prepare data to be rendered by chart.js uring `createChartDataset` function:

```javascript
/**
 * Transforms API response into an object that can be used in the {@link updateChart} call. 
 * @param {Object} data 
 * @param {string} selector 
 */
function createChartDataset(data, selector) {
    let series = data.messages.map(item => item[selector]);
    return {
        'dates': data.messages.map(item => item.timestamp),
        'data': series,
        'min': Math.min.apply(series),
        'max': Math.max.apply(series)
    };
}
```

Finally, let's look at how to update the existing chart's data:

```javascript
/**
 * Updates given chart using data and sets y axis min and max values 
 * @param {ChartJS object} chart 
 * @param {Object} data 
 * @param {number} min 
 * @param {number} max 
 */
function updateChart(chart, data, min, max) {
    chart.data.datasets[0].data = data.data;
    chart.data.datasets[0].labels = data.dates.map(item => moment(item.timestamp).format('ddd HH a'));
    chart.options.scales.yAxes[0].ticks.suggestedMin = min;
    chart.options.scales.yAxes[0].ticks.suggestedMax = max;
    chart.update();

}
```

That is everything that we neet to fetch and render our weather station measurements using an API. Now, let's switch to the styling part and make things look a bit prettier.

## Styling

In this project, we will use [SASS](https://sass-lang.com) extension language as a handy abstraction upon regular CSS. It allows to write stylesheets in a more clean and readable manner. I won't do a detailed walkthrough of the stylesheet and encourage you to clone the repository, open the `index.html` file in the browser and play with the styles yourself. Page styling and design are better to be learned in practice: install `sass`, read documentation and experiment.

# The end

By this post, we conclude the journey that led us from building ESP32 based weather station hardware, writing firmware, coding the backend service that tracks all measurements, building a Telegram notification bot and creating a simple web dashboard for browsing historical data. I hope that this post shed more light into the end-to-end process of making hardware & software to solve a specific problem in mind. Now that you have covered every part of the process, the next step is to go on and create something new yourself! 😉