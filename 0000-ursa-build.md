- Feature Name: ursa-build
- Start Date: 2019-01-10
- RFC PR:
- Ursa Issue:
- Version: 1

# Summary
[summary]: #summary

This RFC covers the design and implementation of the Hyperledger Ursa build
system. It is important to point out that this is just the process by which
build configurations are loaded and the data used to direct the build of Ursa
so that the end result contains only the desired features and implementations.
The design for the Ursa configuration tool is specified in the
[0000-ursa-cfg](https://github.com/hyperledger/ursa-rfcs/0000-ursa-cfg.md) RFC.

# Motivation
[motivation]: #motivation

The Hyperledger Ursa crypto library is designed to provide a uniform
abstraction of many different cryptographic primitives and implementations. As
such, the library is comprised mostly of existing crypto libraries along with
original Rust code that provides the abstraction. Building a library structured
like Ursa requires a sophisticated build system that not only knows how to
build the Rust code but also the underlying crypto libraries that are written
in other programming languages and then to link them all together into a single
library for use by client software.

In addition to being capable of building many different kinds of software, the
build system must also be able to translate and pass along configuration data
to the build processes so that the resulting pieces contain only the features
and implementations desired by the user of Ursa. To do this effectively,
careful attention must be paid to the build process the project settles on. The
rest of this document details the design of the build system the project uses.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Hyperledger Ursa uses the Cargo build tool to manage configuration and building
of the Ursa library. Cargo is used to read in a build configuration and to
drive the build process to completion. It will run the build steps for all
3rd party crypto libraries that are included in an Ursa build, as well as the
necessary abstraction code that provides the uniform API for client software.

For a more in depth understanding of Cargo, 
[the Cargo Book](https://doc.rust-lang.org/cargo/index.html) is the definitive
reference on all of its features. Where possible, references are provided to
relevent sections of the Cargo Book so that readers may seek a more thorough
explanation of a particular aspect of the Ursa build process.

To work with Ursa, you must first install the Rust toolchain along with the
Cargo build tool. The easiest way to do that is to follow the steps outlined
on the [Rustup](https://rustup.rs) web page. For Linux users, simply open your
terminal window and type the following:

```
$ curl https://sh.rustup.rs -sSf | sh
```

When the process is complete, the latest stable release of the Rust Cargo tools
is installed in your home directory and your PATH variable is updated so that
you can run both `rustc` and `cargo` from your terminal. Test it out now. At
the time of this writing, checking the version of both tools looks like this:

```
$ rustc --version
rustc 1.31.1 (b6c32da9b 2018-12-18)

$ cargo --version
cargo 1.31.0 (339d9f9c8 2018-11-16)
```

To build Ursa, you must switch to your local copy of the Ursa source code and
run the command `cargo build`. That instructs cargo to build Ursa using the
default configuration that contains the default set of features and primitives.
To execute the unit tests, execute the command `cargo test`.

Cargo differs from other build tools because it does more than just blindly
follow build steps like make or cmake. Cargo seeks to better understand the
environment in which the build is taking place as well as the environment in
which the resulting binary will be executed. It has sophisticated features for
delegating build steps to build scripts written in Rust as well as other build
tools required for building code not written in Rust. The Ursa build system
must take the specified configuration and not only select which software gets
built, but then also make Cargo run the build tools in the 3rd party libraries
to produce the desired result.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The Ursa build system relies heavily on the use of [Cargo build
scripts](https://doc.rust-lang.org/cargo/reference/build-scripts.html) to do
a lot of the hard work. The sole input to the Ursa build system is an Ursa
build configuration file. The goal of the build configuration file is to
contain all of the information necessary to make a build of Ursa repeatable
such that the results of the build are the same no matter who executes them.
NOTE: This is not to say that it is a [reproducible
build](https://en.wikipedia.org/wiki/Reproducible_builds) system but a key
building block of one. More on that later.

## Build Configuration Files

The core concepts to understand right now is that all inputs to the build
process are taken from the specified build configuration file and as a result
of the build process, an output build configuration file must be created that
contains the exact information from the input build configuration file as well
as any meta data required by more comprehensive reproducible builds systems.
The purpose of the output build configuration file is to provide a diagnostic
tool for examining the inputs to the build process as well as to make possible
moving to a fully-reproducible and deterministic build system in the future.

The build configuration file must contain all of the input information needed
to build Ursa with any/all of the available primitives and features. To
remain consistent with the established Rust tradition, the build configuration
file uses the [TOML format](https://github.com/toml-lang/toml). For anybody
familiar with Rust and Cargo, you are already familiar with TOML. It is the
format used for Cargo.toml files. And in that tradition, all Ursa build
configuration files will have the .toml file extension. Ursa includes a number
of pre-defined build configuration files that are stored in the repo root
sub-folder named `config`.

There is no naming convention for build configuration files other than make the
names as understandable as possible. For example, the default build
configuration file is called `default.toml`. A build configuration that builds
Ursa with just US Government approved cryptography should be called
`us-govt-approved.toml`.

It is important to point out that there is no provision for composing a build
configuration from multiple build configuration files. It is not possible to
"include" one build configuration file from another. This requires builders of
Ursa to first copy a build configuration they want to start from and then make
the necessary changes or to use the [Ursa config
tool](https://github.com/hyperledger/ursa-rfcs/0000-ursa-cfg.md) to load an
existing configuration file, modify it, and save the modified configuration as
a new build configuration file.

## Ursa Custom Cargo Command

Because Ursa uses the workspace feature to organize the sub-libraries
logically, we cannot have a top-level build script to drive the whole process.
Currently, Cargo does not support a workspace build script so we are forced to
develop a [custom Cargo
sub-command](https://doc.rust-lang.org/cargo/reference/external-tools.html?highlight=subcommand#custom-subcommands)
to handle it for us. Custom Cargo sub-commands are installed locally using the
`cargo install` command and they can be executed like the other Cargo commands.
If we call our command `cargo-foo`, once it is installed, it can be executed
by issuing the command `cargo foo` in your terminal.

The proposed name for the Ursa build command is cargo-texo.
The word Latin word "texo" means "build" or "weave". Running the build process
for Ursa is done by executing `cargo texo [<configuration>]`. If no
configuration is specified, the `texo` command will select the configuration
named `default.toml`.

### Cargo-texo Usage

The `cargo-texo` sub-command have a very minimal interface. For now, it only
supports specifying the configuration to build, but later versions will support
specifying the output build configuration file. The usage message shown when
running `cargo texo --help` looks like:

```
cargo-texo
Compile a local package or workspace and all of its dependencies with the
configuration specified in a build configuration TOML file.

USAGE:
    cargo texo [OPTIONS] [<configuration>]

OPTIONS:
    -h, --help                        Prints help information

Cargo-texo will search the a subfolder of the current folder named `configs`
and then the current folder for the specified configuration file. Configuration
files have the .toml file extension but including the extension is optional. To
select the configuration named `foo-bar`, you may specify `foo-bar` or
`foo-bar.toml`. If no configuration is specified, Cargo-texo will search for
`default.toml` to use.
```

As stated in the usage text, `cargo-texo` will search a subfolder named
`configs` and then the current folder for the specified configuration. If no
configuration is specified, it will search for a configuration file named
`default.toml`.

### Driving Cargo

The `cargo-texo` command loads the specified build configuration and uses it
to set up a clean environment to execute the Cargo build tool with the correct
settings. Not only does it execute Cargo but it also gathers output from Cargo
that it processes to create the output build config file from. To accomplish
this, `cargo-texo` will use the `CARGO` environment variable to get the path
to the cargo executable and then execute it as a sub-process. The stdout and
stderr outputs from the cargo command is processed looking for status messages
intented for `cargo-texo` to process.

The Cargo build tool supports outputting messages from build scripts and build
steps on stdout. When a build script outputs `warning=MESSAGE` on stdout, Cargo
echos the message on its stdout as "warning: MESSAGE". By using build scripts
and warnings, specially formatted messages can be passed back to `cargo-texo`
for processing.

Cargo has already established a format for this kind of behavior that is
documented in the [Cargo section on build
scripts](https://doc.rust-lang.org/cargo/reference/build-scripts.html#outputs-of-the-build-script).
Warnings that begin with `texo:` are processed by `cargo-texo` and the primary
means by which information gets recorded in the build output script.

## Ursa Build Configuration File Format

TODO:
* [ ] Specify the section that contains the toolchain configuration including
      the version of rust and cargo and gcc/llvm as well as the target tuple
      and the build machine tuple.
* [ ] Specify the section that defines packages need to be built and which need
      to be excluded.
* [ ] Specify how features and environment variables can be specified for each
      package.
* [ ] Specify how build details are added to the output build configuration.

# Drawbacks
[drawbacks]: #drawbacks

This design may be overly complicated. Adding another layer around cargo may be
a sub-optimal design. As great as Cargo is, it doesn't have the features
necessary for making build configurations easy to specify and repeat.

# Rationale and alternatives
[alternatives]: #alternatives

This design minimizes the chances of a user building Ursa in an incorrect
configuration by supporting pre-defined configurations while also creating a
flexible system for defining new configurations. Cargo is an extremely
powerful build tool and by extending it with a custom command we can make it
ideal for supporting our CI/CD plans as well as our plans for implementing
reproducible builds.

Since the new code in Ursa is written in Rust, Cargo is the obvious choice for
the build tool. It is the native build tool for Rust projects and has all of
the features we need for the complex build tasks required to build Ursa.

# Prior art
[prior-art]: #prior-art

Due to how new Rust and Cargo are, there is very little prior art. The only
major project written in Rust that uses the workspace system is
[Serde](https://github.com/serde-rs/serde) and they don't try to
include/exclude packages from the workspace build.

# Unresolved questions
[unresolved]: #unresolved-questions

- Is it necessary to create a custom Cargo sub-command to support build
  configuration files?
- Is the `taxo:` warning output the best way for build scripts to add build
  configuration data to the output build configuration file?

# Changelog
[changelog]: #changelog

- [10 Jan 2019] - v1 - Initial draft of the build system design.
