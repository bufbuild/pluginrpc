# pluginrpc

[![Build](https://github.com/bufbuild/pluginrpc/actions/workflows/ci.yaml/badge.svg?branch=main)](https://github.com/bufbuild/pluginrpc/actions/workflows/ci.yaml)
[![BSR](https://img.shields.io/badge/BSR-Module-0C65EC)](https://buf.build/bufbuild/pluginrpc)
![License](https://img.shields.io/github/license/bufbuild/pluginrpc)
[![Slack](https://img.shields.io/badge/Slack-Buf-%23e01563)](https://buf.build/links/slack)

PluginRPC is an Protobuf RPC framework for plugins. It enables writing plugins defined by Protobuf
services while only relying on CLI primitives such as args, stdin, and stdout. PluginRPC has no
reliance on network calls. Individual RPCs are invoked via arguments passed to the CLI plugin,
requests are sent on stdin, and responses are sent on stdout. It's everything you need to use
Protobuf services to represent your plugin API without any of the cruft.

PluginRPC has the following goals:

- Make it possible to evolve both the PluginRPC protocol and the APIs for individual plugins in
  backward-compatible ways over time.
- Make it possible to invoke multiple methods as part of a plugin. Many plugin frameworks only
  provide for a single entrypoint. In contrast, PluginRPC provides the full power of Protobuf
  services: multiple services, and multiple RPCs per service, can be exposed.
- Expose the plugin interface in CLI-idiomatic ways. PluginRPC takes JSON-encoded bodies on stdin as
  requests, and sends back JSON-encoded bodies on stdout as responses. PluginRPC exposes individual
  RPCs as sub-commands, optionally allowing the specific sub-commands to be customizable.
- Make it easy to call plugins even in the absence of an official language-specific library.

PluginRPC currently has one official language library:

- [pluginrpc-go](https://github.com/bufbuild/pluginrpc-go).

Golang is a natural language to write plugins in, and we have a direct use case for an RPC library
in Golang for our upcoming custom lint and breaking change plugins in
[buf](https://github.com/bufbuild/buf). However, PluginRPC is purposefully designed to be simple to
implement in any language. If there is sufficient demand, we may provide official implementations
for other languages in the future.

If using Protobuf to write plugins, there's traditionally been two mechanisms:

1. Have a single request message type sent over stdin, and return a single response message type
   over stdout. This is how
   [`protoc`](https://github.com/protocolbuffers/protobuf/blob/main/src/google/protobuf/compiler/plugin.proto)
   operates, for example, taking in a `CodeGeneratorRequest` and sending back a
   `CodeGeneratorRequest` over stdout. This is very simple, however it makes doing anything else
   extremely difficult. All plugin API evolution needs to remain within these single messages for
   all time, and providing functionality for multiple methods is awkward at best (for example, via
   use of `oneofs`).
2. Bring a full-powered network RPC framework into the mix. This is how
   [go-plugin](https://github.com/hashicorp/go-plugin) operates, for example, making a plugin expose
   a gRPC service, and then doing an exec/kill dance to bring the plugin up temporarily and expose
   it on a port, call the required method, and then kill the plugin entirely. While effective in
   allowing lots of flexibility, it's bringing in a very complicated framework to solve a simple
   engineering problem, adding brittleness, requiring network calls, and not allowing plugins to
   remain idiomatic CLI tools

PluginRPC provides the best of both worlds: simple CLI constructs with all the power you require. If
you want to use Protobuf services across the network, use [ConnectRPC](https://connectrpc.com). If
you want to use Protobuf services to write plugins, use PluginRPC.

## Protocol

A plugin is implemented as an executable binary. Plugins may be implemented on sub-commands of the
binary. For example, given the binary `acme`, a plugin author may choose to implement the plugin
under the sub-command `acme plug`. PluginRPC frameworks will provide clients the ability to specify
this sub-command (in [pluginrpc-go](https://github.com/bufbuild/pluginrpc-go), this is done via
`ExecRunnerWithArgs`).

We'll refer to the executable and subsequent base args as `$PLUGIN` below. In our example here,
`acme plug` is what we refer to as `$PLUGIN`. We'll also assume that the plugin wants to implement
the following Protobuf service:

```proto
package acme.foo.v1;

service FooService {
    rpc Bar(BarRequest) returns (BarResponse);
    rpc Baz(BazRequest) returns (BazResponse);
}

message BarRequest {
    string id = 1;
}

message BarResponse {
    string bar = 1;
}

message BazRequest {}

message BazResponse {
    string baz = 1;
}
```

All plugins must respond to two flags:

- `$PLUGIN --plugin-protocol`. This must print the PluginRPC protocol version to stdout with any
  number of subsequent newlines and exit with code `0`. The only protocol version right now is `1`,
  so plugins must print `1` to stdout (or `1\n`, `1\n\n`, etc).

Plugin callers can use `--plugin-protocol` to understand which PluginRPC protocol version the plugin
supports. In the future, if there are newer backwards-incompatible versions of the PluginRPC
protocol, these will be exposed as versions `2`, `3`, etc.

- `$PLUGIN --plugin-spec`. This must print a JSON-encoded
  [`buf.pluginrpc.v1.Spec`](https://buf.build/bufbuild/pluginrpc/docs/main:buf.pluginrpc.v1beta1#buf.pluginrpc.v1beta1.Spec)
  to stdout and exit with code `0`.

Here's an example response to `--plugin-spec` on stdout:

```json
{
  "procedures": [
    {
      "path": "/acme.foo.v1.FooService/Bar",
      "args": ["bar"]
    },
    {
      "path": "/acme.foo.v1.FooService/Baz",
      "args": ["baz"]
    }
  ]
}
```

This specifies that:

- The `Bar` RPC is invoked by calling `$PLUGIN bar`.
- The `Baz` RPC is invoked by calling `$PLUGIN baz`.

The `args` field is optional. If `args` are not specified, a procedure can be invoked with
`$PLUGIN <path>`. For example, if `args` were not specified for `Baz`, then
`$PLUGIN /acme.foo.v1.FooService/Baz` would invoke the `Baz` RPC.

An RPC is invoked by sending a JSON-encoded
[`buf.pluginrpc.v1.Request`](https://buf.build/bufbuild/pluginrpc/docs/main:buf.pluginrpc.v1beta1#buf.pluginrpc.v1beta1.Request)
to stdin. The `body` will contain the RPC request type. Plugins must support nothing being written
to stdin, which will be interpreted as a request with an empty message.

An RPC will respond on stdout with a JSON-encoded
[`buf.pluginrpc.v1.Response`](https://buf.build/bufbuild/pluginrpc/docs/main:buf.pluginrpc.v1beta1#buf.pluginrpc.v1beta1.Response)
and exit with code `0` unless there is a system error. Responses may contain either or both of a
`body`, containing the RPC response type, and/or an `error`, which is a structured
[`buf.pluginrpc.v1.Error`](https://buf.build/bufbuild/pluginrpc/docs/main:buf.pluginrpc.v1beta1#buf.pluginrpc.v1beta1.Error).
If there is a system error, stdout is undefined and will not be processed.

A plugin may always write to stderr as it wishes.

Currently, there is no mechanism for headers, however this will likely be added pre-v1.0.

## Status: Alpha

This framework is in active development, and should not be considered stable. We're publishing it
publicly to get early feedback as we approach stability.

## Legal

Offered under the [Apache 2 license](https://github.com/bufbuild/pluginrpc/blob/main/LICENSE)
