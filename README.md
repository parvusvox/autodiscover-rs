# autodiscover-rs

autodiscover-rs implements a simple algorithm to detect peers on an
IP network, connects to them, and calls back with the connected stream. The algorithm supports both UDP broadcast and multicasting.

## Why the fork?
In my usecase creating Elix, the file transfer utility, I needed the peer discovery to return an address so that Elix can keep trying to connect to the peer while the peer is starting its own server.

## Usage

Cargo.toml
```toml
autodiscover-rs = {git="https://github.com/parvusvox/autodiscover-rs"}
```

In your app:

```Rust
use std::net::{TcpListener, TcpStream};
use std::thread;

use autodiscover_rs::{self, Method};
use env_logger;

fn handle_client(stream: std::io::Result<SocketAddr>) {
    println!("Got a connection from {:?}", stream.unwrap());
}

fn main() -> std::io::Result<()> {
    env_logger::init();
    // make sure to bind before announcing ready
    let listener = TcpListener::bind(":::0")?;
    // get the port we were bound too; note that the trailing :0 above gives us a random unused port
    let socket = listener.local_addr()?;
    thread::spawn(move || {
        // this function blocks forever; running it a seperate thread
        autodiscover_rs::run(&socket, Method::Multicast("[ff0e::1]:1337".parse().unwrap()), |s| {
            // change this to task::spawn if using async_std or tokio
            thread::spawn(|| handle_client(s));
        }).unwrap();
    });
    let mut incoming = listener.incoming();
    while let Some(stream) = incoming.next() {
        // if you are using an async library, such as async_std or tokio, you can convert the stream to the
        // appropriate type before using task::spawn from your library of choice.
        thread::spawn(|| handle_client(stream));
    }
    Ok(())
}
```

## Notes

The algorithm for peer discovery is to:
- Send a message to the broadcast/multicast address with the configured 'listen address' compressed to a 6 byte (IPv4) or 18 byte (IPv6) packet
- Start listening for new messages on the broadcast/multicast address; when one is recv., connect to it and run the callback

This has a few gotchas:
- If a broadcast packet goes missing, some connections won't be made


### Packet format

The IP address we are broadcasting is of the form:

IPv4:

```Rust
buff[0..4].clone_from_slice(&addr.ip().octets());
buff[4..6].clone_from_slice(&addr.port().to_be_bytes());
```

IPv6:

```Rust
buff[0..16].clone_from_slice(&addr.ip().octets());
buff[16..18].clone_from_slice(&addr.port().to_be_bytes());
```

## TODO

1. Provide features for async frameworks (such as async_std and tokio)
2. Figure out some way of testing this
3. Provides some mechanism to stop the thread listening for discovery
