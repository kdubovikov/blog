---
layout: post
title:  "Async Unicorns love Rust"
date:   2020-07-27 00:00:00
categories: rust
---

Before going on to write a backend for our weather station we first need to familiarize ourselves with a few concepts from the Rust world. If you are unfamiliar with the language take a few minutes to read through [Learn Rust in Y minutes](https://learnxinyminutes.com/docs/rust/) to get used to the syntax.

When we will write a Telegram bot with Rust, we will use a technique called asynchronous programming. Let's tackle what that means.

<!--more-->

Previous post in the series: [Coding For ESP32]({% post_url 2020-06-21-build-yourself-a-weathe-station-part-2 %})

## Selling unicorns. Synchronously

Our bot will use a paradigm called asynchronous input-output, so we will need to understand what is standing behind those words. To understand this, let's first define what we mean by synchronous input/output.

Imagine we need to get the contents of a web page into our program. For example, let's suppose that we want to write an app that allows users to buy unicorns using a command-line interface 🦄. Turns out that there are a lot of techies in the unicorn-lover community 👨🏻‍💻, so the investors bought in and agreed to finance the project. 

{% maincolumn 'assets/img/esp32-weather-station/post-3-rust-async/unicorn.jpg' %}

We have partnered with one of the best online shops who has given us an address of the best deals page: [`http://unizon.com/best-deals`](http://unizon.com/best-deals). So, we need to fetch the data from this page. The most straightforward way to do this would be to find some HTTP client library that implements low-level details of fetching the web page via HTTP protocol:

```rust
// Initialize our HTTP client library. 
// The one we use here does not really exist, 
// so consider the code below as pseudocode
let client = Client::new();

// Get the response
let resp = client.get("[http://unizon.com/best-deals](http://unizon.com/best-deals)");

println!("Response: {}", resp.body());
```

Under the hood this code does the following:

{% maincolumn 'assets/img/esp32-weather-station/post-3-rust-async/Sync_HTTP_request.svg' %}

Here, `client.get` is a blocking call. This means that our main application `Buy'a'Unicorn App` needs to wait while lower-level components fetch the request.

This overhead may be acceptable, but imagine that we will start making thousands of requests. Then, most of the runtime our app will wait while numbers are running back and forth over the wires.

Now, suppose that our userbase has grown and we have decided that we need to get more partners on board. We have found a few dozen of other shops that joyfully agreed to join our platform. The functionality of our app has expanded: we now have price lists, filters, sortings, and many other exciting command-line options. This means more work inside the main app, and more requests to other 🦄 shops. 

We realize that our app does not scale anymore. The more new shops we add, the more we will need to wait for network I/O. This means we need to escape the bottleneck if we want our little unicorn business to grow.

# Tangling Threads

One way to do this would be to introduce threads. Threads are lightweight (but not free) concurrency primitives that allow you to execute parts of your appreciation in a non-blocking manner. This means that we can use them to issue multiple HTTP requests to unicorn shops in parallel and automatically update data as it is coming in. 

Before continuing, let's first tackle what stands behind parallelism and concurrency to make things more clear. CPU at your computer har multiple mini-CPUs inside called cores. Those cores allow executing multiple low-level instructions in parallel. Multi-core execution on a 4-core CPU will allow us to execute 4 tasks independently of each other. But let's see how many parallel processes my laptop can handle:

```bash
$ ps aux | wc -l
     875
```

That's a lot more than 4, right? How is this possible? If you use Intel CPUs, then most of them will have a hyper-threading option available, which will separate each physical CPU core into two logical cores which can be used by your operating system. 

Doubling the core number is still not sufficient to run 875 processes, so there is more to it than hyperthreading. It turns out that most of the time applications you run wait for something: user input, hard drive, network communication. Operating systems leverage this fact to fake parallelization by introducing a thing called concurrent execution.

{% maincolumn 'assets/img/esp32-weather-station/post-3-rust-async/concurrency.jpg' %}

Two processes are executing concurrently if they can execute in overlapping periods of time. However, it does not mean that they execute their instructions at the same time. For example, one concurrent process can pause and let the other run. And when two processes execute their instructions simultaneously they are called to be run in parallel.

Operating systems provide complicated schedulers that automatically control which processes execute in parallel, which concurrently and how exactly they should switch between each other. In practice, concurrent execution happens so fast that for us it looks like all of the processes are running in parallel.

Next, let's clarify the difference between two related concepts: **processes** and **threads**. 

The process represents an executing program in an operating system. Each process is completely isolated from other processes. The only way one process can communicate to another is by utilizing some external shared resources. For example, a file system, or a database server. Due to their properties, processes have some overhead. Creating and maintaining processes takes time, and switching between them requires work too.

{% maincolumn 'assets/img/esp32-weather-station/post-3-rust-async/Process_communication.svg' %}

Let's look at how we can use processes in Rust:

```rust
use std::process::Command;

// execute echo command and save it's output to the variable
let output = Command::new("echo")
                     .arg("Hello world")
                     .output()
                     .expect("Failed to execute command");
assert_eq!(b"Hello world\n", output.stdout.as_slice());

// in this way, we could fetch data using external applications
let response = Command::new("curl")
                     .arg("http://unicorn.shop/store/items")
                     .output()
                     .expect("Failed to execute command");

```

If our unicorn shop app needs to communicate with tens or hundreds of different shops, then processes might solve our issue. However, if we will scale to thousands of shops (for which I have absolutely no doubt), then processes will start to feel show and sluggish.

The more lightweight concept of threads comes to our rescue. Think of them as a set of lightweight mini-processes inside your large process. Threads are much faster to create and destroy. Also, they share memory access, so you won't need to use expensive I/O-based communication protocols just to share information inside your own application. Howewer, all this power comes with a grain of salt.

{% maincolumn 'assets/img/esp32-weather-station/post-3-rust-async/salt-mine.jpg' %}

Shared memory access complicates the programmer's job by a lot. It introduces notoriously hard to debug problems such as data races and deadlocks. Thankfully, Rust takes away most of that pain. The borrow checker introduces strict mutability control that allows the Rust compiler to ensure that two different threads do not share simultaneous access to a single variable. Of course, there are ways to circumvent this, but it takes a lot of conscious effort to break through Rust's safety net. Let's see how we can manipulate threads in Rust:

```rust
use std::thread;
use std::sync::mpsc::channel;

// MPSC stands for Multiple Producer Single consumer.
// Channels are a useful abstraction for communication between threads:
// tx (transmitter) allows multiple producers to send data over the channel,
// rx (receiver) allows a single consumer to recieve the data from producers.
// In our case, tx will be used by multiple threads to send responses to our reciever thread
let (tx, rx) = channel();

let sender = thread::spawn(move || {
		// We will use an imaginary API for sending HTTP GET requests for simplification
		let response = get_http_response(String::from("http://unicorn.shop/store/items"))
    tx.send(response)
        .expect("Unable to send on channel");
});

let receiver = thread::spawn(move || {
    let value = rx.recv().expect("Unable to receive from channel");
    println!("{}", value);
});

sender.join().expect("The sender thread has panicked");
receiver.join().expect("The receiver thread has panicked");
```

Creating a thread each time we need to perform can still be costly, so we can introduce another useful tool: a thread pool. Thread pool herds a number of threads and gives you a free one each time you need it:

```rust
// Rust standard library does not provide a thread pool implementation,
// so we will a `threadpool` crate.
// https://docs.rs/threadpool/1.8.1/threadpool/

use threadpool::ThreadPool;
use std::sync::mpsc::channel;

// Create a thread pool with 4 worker threads ready to handle our requests
let n_workers = 4;
let pool = ThreadPool::new(n_workers);

// Our shop URLs
let shops = vec![String::from("http://unicorn.shop/store/items"), 
								 String::from("http://all-unicorns.ml/inventory")];

// Issue each request in a thread pool.
// The for loop won't block on `pool.execute` call
let (tx, rx) = channel();
for shop_url in shops.iter() {
    let tx = tx.clone();
    pool.execute(move|| {
				// Again, using our fictionary API to send requests to
				// imaginary unicorn shops :)
				let response = get_http_response(shop_url)
        tx.send(response).expect("channel will be there waiting for the pool");
    });
}

// We can do other stuff without waiting for responses
some_important_work_to_do();

// Here, the loop will print request in the order 
// they arrive from the shop servers
while let Some(response) = rx.iter() {
	println!("{}", response);
}
```

Now that you know how to use threads, let's fast-forward our unicorn business for a few years. We are still growing, and now almost all shops are integrated into our platform. Someday, unicorn appliances and care products sellers got interested in providing co-sell options so that you won't only buy a unicorn, but a fully packed experience with the best saddle and latest horn care shampoo on the market.

All this means that we will have a great expansion of our service, and a lot more HTTP requests to do. Also, our application starts to become more complex. New additions to the business logic  introduce lots of buds and are slower to implement. The threading backend that we use won't keep up with the job anymore. 

# Async to the rescue

To solve our problems, we must shift the paradigm. Up to this point, we thought about our application in two parts: the main thread that handles all of the application logic and several worker threads in a thread pool that handles networking I/O for us.

But why do we need to switch threads in the first place? Why code in our main thread can't continue executing while some other part of the application is waiting for an I/O? In fact, every major operating system provides an API to handle off the I/O work to the OS kernel and receive notifications when data is available for reading. 

This functionality can be leveraged to avoid blocking on the I/O code in a single thread: if your application encounters an I/O call it can continue executing other tasks while waiting for the data to become available {% sidenote 1 "In Linux, this functionality is available via `epoll` API"%}. In reality, this turns out to be a fairly complex task that can be implemented with the event loop architecture pattern.

{% maincolumn 'assets/img/esp32-weather-station/post-3-rust-async/Event_loop.svg' %}



As you can see, the event loop operates by observing events and executing tasks. A task can be thought of as an abstraction over a block of code. The event loop works by observing I/O events from the operating system and executing tasks that read or write data. 

This model introduces a significant drawback compared to the traditional approach: the event loop runs in a single thread. This means that if our tasks contain anything but I/O operations their execution will block the entire event loop. That leads to a conclusion: if your application uses a lot of I/O requests, it will get a big performance boost from using the event loop approach. However, if it is CPU-heavy and uses lots of computational resources, it will block the event loop too much for it to provide any benefit. Modern asynchronous libraries provide a way to fight this by providing additional thread pool which can be used to execute blocking tasks without stopping the event loop.

Another important part of the solution is the asynchronous programming paradigm. Modern programming languages simplify working with asynchronous code by providing special syntax constructs. For example, Rust allows us to define `async` closures and functions. 

```rust
// `block_on` blocks the current thread until the provided future has run to
// completion. Other executors provide more complex behavior, like scheduling
// multiple futures onto the same thread.
use futures::executor::block_on;

async fn hello_world() {
    println!("hello, world!");
}

fn main() {
    let future = hello_world(); // Nothing is printed
    block_on(future); // `future` is run and "hello, world!" is printed
}
```

The above example is from from the [Asynchronous Programming In Rust](https://rust-lang.github.io/async-book/01_getting_started/04_async_await_primer.html). I suggest reading the book for a more in-depth intro into async Rust.

Each asynchronous function returns a `Future`, a special construct that designates a result, which will be returned sometime in the future. Notice that the `hello_world` function will not be executed until we `block_on` the future.

In practice, we will work with a [tokio](https://docs.rs/tokio/) crate, which implements the event loop pattern and wraps `mio` library that uses the underlying operating system's asynchronous I/O implementatein and wraps it in a convenient interface. Another important library we will use in the ESP weather station project is [tbot](https://tbot.rs), a Telegram bot library built on top of `tokio`. Let's look at how to build a simple bot with it:

```rust
use tbot::prelude::*;
use tokio::sync::Mutex;

#[tokio::main]
async fn main() {
	// We create the bot's event loop by using token from the environment variable BOT_TOKEN
  let mut bot = tbot::from_env!("BOT_TOKEN")
    .event_loop();

	// next, we register a command which greets the user with a predefinend message
  bot.command("say_hi", |context| {

		// this construct allows us to move variables from the enclosing environment into
    // the closure below
    async move {
      let message = String::from("Hi there!");
			
			// finally, we issue a blocking call that sends the message back to the user.
			// await.unwrap() allows us to force the execution of a send_message API call.
			// without it, nothing would happen
      context
        .send_message(&message)
        .call()
        .await
        .unwrap();
    }
  });
	
	// here, we start the Telegram API poller which will periodically ask the server about new events
  // such as incoming messages
  bot.polling().start().await.unwrap();
}
```

Notice the `#[tokio::main]` macro. According to the documentation, it "marks async function to be executed by selected runtime". By runtime we mean the underlying implementation of tokio's scheduler that executes tasks. As defined in the framework's documentation, the runtime consists of the following components:

* An I/O event loop, called the driver, which drives I/O resources and dispatches I/O events to tasks that depend on them.
* A scheduler to execute tasks that use these I/O resources.
* A timer for scheduling work to run after a set period of time.

# Wrapping up

Now, we can scale our small unicorn shop to be a global bot-powered marketplace where buyers and sellers can satisfy their unicorn needs 24/7, worldwide by using an asynchronous programming model. Apart from that, that's all we needed to cover to start developing a chat bot for our weather statio.

We have looked at three ways of dealing with I/O in our application:

1. Synchronous I/O, where everything happens in the same thread
2. Threaded I/O, where we offload I/O work to a different thread by paying a cost of creating threads and switching between them
3. Asynchronous I/O, where we use the event loop pattern to offload the I/O work to the operating system and receive notifications via special APIs such as linux's `epoll` when our I/O requests are done

While providing extremely fast I/O, asynchronous paradigm is not without its drawbacks:

- It is harder to read, write and maintain async coode
- Async I/O benefits diminish with the amount of CPU-heavy computations you need to make

In the [next post](https://blog.kdubovikov.ml/articles/rust/building-a-weather-station-bot), we will be looking at how to build a Telegram bot for our weather station using Rust as a programming language, Tokio as an async I/O library and tbot as a Telegram bot framework.
