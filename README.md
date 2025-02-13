# Rust gRPC Demo

A Demo for GRPC Server With Rust.

from blog [Building a gRPC Server With Rust](https://yzhong52.github.io/2022/05/01/rust-grpc.html)

## Background and Introduction

### RPC vs JSON vs SOAP

Once I learn about [gRPC](https://grpc.io/) and [Thrift](https://github.com/facebook/fbthrift), it is hard to go back to using the more transitional JSON-based REST API or [SOAP](https://en.wikipedia.org/wiki/SOAP) API.

The two well-known [RPC](https://en.wikipedia.org/wiki/Remote_procedure_call) frameworks, gRPC, and Thrift, have many similarities. The former originates from Google while the latter originates from Facebook. Both of them are easy to use, have great support for a wide range of programming languages, and both are performant.

The most valuable feature is code generation in multiple languages and also server-side reflection. These make the API essentially type-safe. With server-side reflection, it makes it much easier to explore the schema definition of the API without having to read and understand the implementation.

### Grpc vs Thrift

[Apache Thrift](https://thrift.apache.org/) historically has been a popular choice. However, in recent years, due to a lack of continuous support from Facebook, and fragmentation with the fork of [fbthrift](https://github.com/facebook/fbthrift), it slowly lost popularity.

In the meantime, gRPC has caught up with more and more features with a healthier ecosystem.

![](images/2022-05-26-10-12-23.png)

Comparison between GRPC (blue) with Apache Thrift (red). [Google Trends](https://trends.google.com/trends/explore?date=all&q=GRPC,%2Fm%2F02qd4d1)

![](images/2022-05-26-10-13-20.png)

GitHub star history between gRPC, fbThrift, and Apache Thrift. [https://star-history.com](https://star-history.com/#grpc/grpc&facebook/fbthrift&apache/thrift&Date)

As of today, unless your application is affiliated with Facebook in someway, there is no good reason to consider Thrift.

### How about GraphQL?

[GraphQL](https://github.com/graphql/graphql-spec) is another framework initiated from Facebook. It shares many similarities with the two RPC frameworks above.

One of the biggest pain points in mobile API development is that some users never upgrade their app. Because we want to maintain backward compatibility, we either have to keep old unused fields in the API or create multiple versions of the API. [One motivation of GraphQL was to solve that problem](https://youtu.be/783ccP__No8). It is designed to be a “query language” and allow the client to specify what data fields that it needs. This makes it easier to handle backward compatibility.

GraphQL has great value in developing mobile APIs as well as public facing APIs (such as [GitHub](https://docs.github.com/en/graphql)). Since, in both cases, we could not easily control the client behavior.

However, if we are building an API for the web frontend or API for internal backend services, there is little benefit of choosing GraphQL over gRPC.

## Rust

Above is a small overview of the networking frameworks so far. Besides networking, we also need to decide on a language for the application server.

Based on [Stack Overflow Survey](https://insights.stackoverflow.com/survey/2021#most-loved-dreaded-and-wanted-language-love-dread): “For the sixth-year, Rust is the most loved language.” Its type-safety, elegant memory management, and wide community support, and performance, all make Rust a very attractive and promising programming language for backend service developments despite the relatively steeper learning curve.

![](images/2022-05-26-10-15-19.png)

Rust is the most loved language. Stackoverflow Survey 2021

We also start seeing wider and wider adoption of Rust in the industry: [Facebook](https://engineering.fb.com/2021/04/29/developer-tools/rust/), [Dropbox](https://www.wired.com/2016/03/epic-story-dropboxs-exodus-amazon-cloud-empire/), [Yelp](https://youtu.be/u6ZbF4apABk), [AWS](https://aws.amazon.com/blogs/opensource/sustainability-with-rust/), [Google](https://opensource.googleblog.com/2021/02/google-joins-rust-foundation.html), etc. It is clear that Rust will continue to grow and is here to stay.

That’s what we’ll look in in today’s tutorial — building a small server with gRPC in Rust.

## Install Rust

Install Rust with the following:

`$ curl --proto '=https' --tlsv1.2 -sSf <https://sh.rustup.rs> | sh`

If you’ve previously installed Rust, we can update it via:

`$ rustup update stable`

Let’s double-check the installed version of rustc (the Rust compiler) and cargo (the Rust package manager):

```sh
$ rustc --version
rustc 1.61.0 (fe5b13d68 2022-05-18)
$ cargo --version
cargo 1.61.0 (a028ae42f 2022-04-29)
```

For more information about installation, checkout https://www.rust-lang.org/tools/install.

## Create a Rust Project

Run the following to create a new “Hello World” project:

`$ cargo new rust_grpc_demo --bin`

Let’s compile and run the program:

```sh
$ cd rust_grpc_demo
$ cargo run
   Compiling rust_grpc_demo v0.1.0 (/Users/yuchen/Documents/rust_grpc_demo)
    Finished dev [unoptimized + debuginfo] target(s) in 1.75s
     Running `target/debug/rust_grpc_demo`
Hello, world!
```

This shows the file structures we have so far:

```sh
$ find . -not -path "./target*" -not -path "./.git*" | sed -e "s/[^-][^\/]*\//  |/g" -e "s/|\([^ ]\)/| - \1/"
  |-Cargo.toml
  |-Cargo.lock
  |-src
  |  |-main.rs
```

## Define gRPC Interface

gRPC uses [Protocol Buffers](https://developers.google.com/protocol-buffers/docs/overview) for serializing and deserializing data. Let’s define the server API in a .proto file.

```sh
$ mkdir proto
$ touch proto/bookstore.proto
```

We define a book store service, with only one method: provide a book id, and return some details about the book.

```proto
syntax = "proto3";

package bookstore;

// The book store service definition.
service Bookstore {
  // Retrieve a book
  rpc GetBook(GetBookRequest) returns (GetBookResponse) {}
}

// The request with a id of the book
message GetBookRequest {
  string id = 1;
}

// The response details of a book
message GetBookResponse {
  string id = 1;
  string name = 2;
  string author = 3;
  int32 year = 4;
}
```

We will create our gRPC service with [tonic](https://docs.rs/tonic/latest/tonic/). Add the following dependencies to the Cargo.toml file:

```toml
[package]
name = "rust_grpc_demo"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
tonic = "0.7.1"
tokio = { version = "1.18.0", features = ["macros", "rt-multi-thread"] }
prost = "0.10.1"

[build-dependencies]
tonic-build = "0.7.2"
```

To generate Rust code from `bookstore.proto`, we use `tonic-build` in the crate’s `build.rs` build-script.

`$ touch build.rs`

Add the following to the `build.rs` file:

```rust
use std::{env, path::PathBuf};

fn main() {
    let proto_file = "./proto/bookstore.proto";

    tonic_build::configure()
        .build_server(true)
        .out_dir("./src")
        .compile(&[proto_file], &["."])
        .unwrap_or_else(|e| panic!("protobuf compile error: {}", e));

    println!("cargo:rerun-if-changed={}", proto_file);
}
```

One thing specific to point out that we added this `.out_dir("./src")` to change the default output directory to the src directory so that we can see the generated file easier for the purpose of this post.

One more thing before we are ready to compile.`tonic-build` depends on the [Protocol Buffers compiler](https://grpc.io/docs/protoc-installation/) to parse .proto files into a representation that can be transformed into Rust. Let’s install protobuf:

`$ brew install protobuf`

And double check that the _protobuf_ compiler is installed properly:

```sh
$ protoc --version
libprotoc 3.19.4
```

Ready to compile:

```sh
$ cargo build
    Finished dev [unoptimized + debuginfo] target(s) in 0.31s
```

With this, we should have a file src/bookstore.rs generated. At this point, our file structure should look like this:

```sh
  | - Cargo.toml
  | - proto
  |  | - bookstore.proto
  | - Cargo.lock
  | - build.rs
  | - src
  |  | - bookstore.rs
  |  | - main.rs
```

## Implement the Server

Finally, time to put the service together. Replace the `main.rs` with the following:

```rust
use tonic::{transport::Server, Request, Response, Status};

use bookstore::bookstore_server::{Bookstore, BookstoreServer};
use bookstore::{GetBookRequest, GetBookResponse};


mod bookstore {
    include!("bookstore.rs");
}


#[derive(Default)]
pub struct BookStoreImpl {}

#[tonic::async_trait]
impl Bookstore for BookStoreImpl {
    async fn get_book(
        &self,
        request: Request<GetBookRequest>,
    ) -> Result<Response<GetBookResponse>, Status> {
        println!("Request from {:?}", request.remote_addr());

        let response = GetBookResponse {
            id: request.into_inner().id,
            author: "Peter".to_owned(),
            name: "Zero to One".to_owned(),
            year: 2014,
        };
        Ok(Response::new(response))
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "[::1]:50051".parse().unwrap();
    let bookstore = BookStoreImpl::default();

    println!("Bookstore server listening on {}", addr);

    Server::builder()
        .add_service(BookstoreServer::new(bookstore))
        .serve(addr)
        .await?;

    Ok(())
}
```

As we can see, for the sake of simplicity, we don’t really have a database of book setup. In this endpoint, we simply return a fake book.

Time to run the server:

```sh
$ cargo run
   Compiling rust_grpc_demo v0.1.0 (/Users/yuchen/Documents/rust_grpc_demo)
    Finished dev [unoptimized + debuginfo] target(s) in 2.71s
     Running `target/debug/rust_grpc_demo`
Bookstore server listening on [::1]:50051
```

Nice we have our gRPC server in Rust up and running!

## Bonus: Server Reflection

As stated at the beginning, I was first impressed by gRPC initially because of its capability of doing server reflection. Not only is it handy during service development, but it also makes communication with frontend engineers a lot easier. So, it wouldn’t be complete to conclude this tutorial without explaining how to add that for the Rust Server.

Add the following to the dependencies:

`tonic-reflection = "0.4.0"`

Update `build.rs`. The lines that need to be changed are marked with // Add this comment.

```rust
use std::{env, path::PathBuf};

fn main() {
    let proto_file = "./proto/book_store.proto";
    let out_dir = PathBuf::from(env::var("OUT_DIR").unwrap()); // Add this

    tonic_build::configure()
        .build_server(true)
        .file_descriptor_set_path(out_dir.join("greeter_descriptor.bin")) // Add this
        .out_dir("./src")
        .compile(&[proto_file], &["."])
        .unwrap_or_else(|e| panic!("protobuf compile error: {}", e));

    println!("cargo:rerun-if-changed={}", proto_file);
}
```

And finally, update the `main.rs` to the following.

```rust
use tonic::{transport::Server, Request, Response, Status};

use bookstore::bookstore_server::{Bookstore, BookstoreServer};
use bookstore::{GetBookRequest, GetBookResponse};


mod bookstore {
    include!("bookstore.rs");

    // Add this
    pub(crate) const FILE_DESCRIPTOR_SET: &[u8] =
        tonic::include_file_descriptor_set!("greeter_descriptor");
}


#[derive(Default)]
pub struct BookStoreImpl {}

#[tonic::async_trait]
impl Bookstore for BookStoreImpl {
    async fn get_book(
        &self,
        request: Request<GetBookRequest>,
    ) -> Result<Response<GetBookResponse>, Status> {
        println!("Request from {:?}", request.remote_addr());

        let response = GetBookResponse {
            id: request.into_inner().id,
            author: "Peter".to_owned(),
            name: "Zero to One".to_owned(),
            year: 2014,
        };
        Ok(Response::new(response))
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "[::1]:50051".parse().unwrap();
    let bookstore = BookStoreImpl::default();

    // Add this
    let reflection_service = tonic_reflection::server::Builder::configure()
        .register_encoded_file_descriptor_set(bookstore::FILE_DESCRIPTOR_SET)
        .build()
        .unwrap();

    println!("Bookstore server listening on {}", addr);

    Server::builder()
        .add_service(BookstoreServer::new(bookstore))
        .add_service(reflection_service) // Add this
        .serve(addr)
        .await?;

    Ok(())
}
```

## Testing The gRPC Server

There are many GUI clients to play with gRPC Server, such as [Postman](https://www.postman.com/), [Kreya](https://kreya.app/), [bloomrpc](https://github.com/bloomrpc/bloomrpc), [grpcox](https://github.com/gusaul/grpcox), etc. To keep things simple for today, we will use a command line tool [grpc_curl](https://github.com/fullstorydev/grpcurl).

To install:

`go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest`

And to test out our first gRPC endpoint:

```sh
$ grpcurl -d '{"id":"1"}'  -plaintext localhost:50051 bookstore.Bookstore/GetBook
{
  "id": "1",
  "name": "Zero to One",
  "author": "Peter",
  "year": 2014
}

$ grpcurl -plaintext localhost:50051 list
bookstore.Bookstore
grpc.reflection.v1alpha.ServerReflection
```

or by grpc_cli: `brew install grpc`

```sh
$ grpc_cli list localhost:50051
bookstore.Bookstore
grpc.reflection.v1alpha.ServerReflection

$ grpc_cli call localhost:50051 bookstore.Bookstore.GetBook "id: 'test-book-id'"
connecting to localhost:50051
Received initial metadata from server:
date : Thu, 26 May 2022 04:24:31 GMT
id: "test-book-id"
name: "Zero to One"
author: "Peter"
year: 2014
Rpc succeeded with OK status
```

Rpc succeeded with OK status

Looks like it works! And that, my friend, is how we build a gRPC server in Rust.

That’s it for today. Thanks for reading and happy coding! As usual, the source code is available on [GitHub](https://github.com/yzhong52/rust_grpc_demo).

## cargo upgrade

[cargo-upgrades](https://crates.io/crates/cargo-upgrades)

```sh
$ cargo install -f cargo-edit
   Installed package `cargo-edit v0.9.1` (executables `cargo-add`, `cargo-rm`, `cargo-set-version`, `cargo-upgrade`)

$ cargo upgrade
    Updating 'https://mirrors.sjtug.sjtu.edu.cn/git/crates.io-index' index
rust_grpc_demo:
    Upgrading prost v0.10.1 -> v0.10.4
    Upgrading tokio v1.18.0 -> v1.18.2
    Upgrading tonic v0.7.1 -> v0.7.2
```
