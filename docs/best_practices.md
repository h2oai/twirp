---
id: "best_practices"
title: "Best Practices"
sidebar_label: "Best Practices"
---

Twirp simplifies service design when compared with a REST endpoint: method
definitions, message types and parsing is handled by the framework (i.e. you
don’t have to worry about JSON fields or types). However, there are still some
things to consider when making a new service in Twirp, mainly to keep
consistency.

## Folder/Package Structure

The recommended folder/package structure for your twirp `<service>` is:
```
/cmd
    /<service>
        main.go
/rpc
    /<service>
        service.proto
        // and auto-generated files
/internal
    /<service>server
        server.go
        // and usually one other file per method
```

For example, for the Haberdasher service it would be:
```
/cmd
    /haberdasherserver
        main.go
/rpc
    /haberdasher
        service.proto
        service.pb.go
        service.twirp.go
/internal
    /haberdasherserver
        server_test.go
        server.go
        make_hat_test.go
        make_hat.go
```

Notes:
 * Keep the `.proto` and generated files in their own package.
 * Do not implement the server or other files in the same package. This allows
   other services to do a "clean import" of the autogenerated client.
 * Do not name the package something generic like `api`, `client` or `service`;
   name it after your service. Remember that the package is going to be imported
   by other projects that will likely import other clients from other services
   as well.

## `.proto` File

The `.proto` file is the source of truth for your service design.

 * The first step to design your service is to write a `.proto` file. Use that
   to discuss design with your coworkers before starting the implementation.
 * Use proto3 (first line should be `syntax="proto3"`). Do not use proto2.
 * Use `option go_package = "<service>";` for the Go package name.
 * Add comments on message fields; they translate to the generated Go
   interfaces.
 * Don’t worry about fields like `user_id` being auto-converted into `UserId` in
   Go. I know, the right case should be `UserID`, but it's not your fault how
   the protoc-gen-go compiler decides to translate it. Avoid doing hacks like
   naming it `user_i_d` so it looks "good" in Go (`UserID`). In the future, we
   may use a better Go code generator, or generate clients for other languages
   like Python or JavaScript.
 * rpc methods should clearly be named with <action><resource> (i.e.: ListBooks,
   GetBook, CreateBook, UpdateBook, RenameBook, DeleteBook). See more in "Naming
   Conventions" below.

The header of the `.proto` file should look like this (change <repo> and
<service> with your values):
```go
syntax = "proto3"

package <organization>.<repo>.<service>;

option go_package = "<service>";
```

## Specifying protoc version and using retool for protoc-gen-go and protoc-gen-twirp

Code generation depends on `protoc` and its plugins `protoc-gen-go` and
`protoc-gen-twirp`. Having different versions may cause problems.

Make sure to specify the required `protoc` version in your README or
CONTRIBUTING file.

For the plugins, you can use [retool](https://github.com/twitchtv/retool). Like
with most Go commands used to manage your source code, retool makes it easy to
lock versions for all team members.

```sh
$ retool add github.com/golang/protobuf/protoc-gen-go master
$ retool add github.com/twitchtv/twirp/protoc-gen-twirp master
```

Using a Makefile is a good way to simplify code generation:

```Makefile
gen:
	# Auto-generate code
	retool do protoc --proto_path=. --twirp_out=. --go_out=. rpc/<service>/service.proto

upgrade:
	# Upgrade glide dependencies
	retool do glide update --strip-vendor
	retool do glide-vc --only-code --no-tests --keep '**/*.proto'
```

## Naming Conventions

Like in any other API or interface, it is very important to have names that are
simple, intuitive and consistent.

Respect the [Protocol Buffers Style Guide](https://developers.google.com/protocol-buffers/docs/style):
 * Use `CamelCase` for `Service`, `Message` and `Type` names.
 * Use `underscore_separated_names` for field names.
 * Use `CAPITALS_WITH_UNDERSCORES` for enum value names.

For naming conventions, the
[Google Cloud Platform design guides](https://cloud.google.com/apis/design/naming_convention)
are a good reference:
 * Use the same name for the same concept, even across APIs.
 * Avoid name overloading. Use different names for different concepts.
 * Include units on field names for durations and quantities (e.g.
   `delay_seconds` is better than just `delay`).

For times, we have a few Twitch-specific conventions that have worked for us:
 * Timestamp names should end with `_at` whenever possible (i.e. `created_at`,
   `updated_at`).
 * Timestamps should be [RFC3339](https://tools.ietf.org/html/rfc3339) strings
   (in Go it's very easy to generate these with `t.Format(time.RFC3339)` and
   parse them with `time.Parse(time.RFC3339Nano, t)`).
 * Timestamps can also be a `google.protobuf.Timestamp`, in which case their
   names should end with `_time` for clarity.

## Default Values and Required Fields

In proto3 all fields have zero-value defaults (string is `""`, int32 is `0`), so
all fields are optional.

If you want to make a required field (i.e. "name is required"), it needs to be
handled by the service implementation. But to make this clear in the `.proto`
file:
 * Add a "required" comment on the field. For example `string name = 1; //
   required` implies that the server implementation will return an
   `twirp.RequiredArgumentError("name")` if the name is empty.

If you need a different default (e.g. limit default 20 for paginated
collections), it needs to be handled by the service implementation. But to make
this clear in the `.proto` file:
 * Add a "(default X)" comment on the field. For example `int32 limit = 1; //
   (default 20)` implies that the server implementation will convert the
   zero-value 0 to 20 (0 == 20).
 * For enums, the first item is the default.

Your service implementation cannot tell the difference between empty and missing
fields (this is by design). If you really need to tell them apart, you need to
use an extra bool field, or use `google/protobuf.wrappers.proto` messages (which
can be nil in go).

## Twirp Errors

Protocol Buffers do not specify errors. You can always add an extra field on the
returned message for the error, but Twirp has an excellent system that you
should use instead:

 * Familiarize yourself with the possible [Twirp error codes](errors.md) and use
   the ones that make sense for each situation (i.e. `InvalidArgument`,
   `NotFound`, `Internal`). The codes are very straightforward and are almost
   the same as in gRPC.
 * Always return a `twirp.Error`. Twirp allows you to return a regular `error`,
   that will get wrapped with `twirp.InternalErrorWith(err)`, but it is better
   if you explicitly wrap it yourself. Being explicit makes the server and the
   client to always return the same twirp errors, which is more predictable and
   easier for unit tests.
 * Include possible errors on the `.proto` file (add comments to RPC methods).
 * But there's no need to document all the obvious `Internal` errors, which can
   always happen for diverse reasons (e.g. backend service is down, or there was
   a problem on the client).
 * Make sure to document (with comments) possible validation errors on the
   specific message fields. For example `int32 amount = 1; // must be positive`
   implies that the server implementation will return a
   `twirp.InvalidArgument("amount", "must be positive")` error if the condition
   is not met.
 * Required fields are also validation errors. For example, if a given string
   field cannot be empty, you should add a "required" comment in the proto file,
   which implies that a `twirp.RequiredArgumentError(field)` will be returned if
   the field is empty (or missing, which is the same thing in proto3). If you
   are using proto2 (I hope not), the "required" comment is still preferred over
   the required field type.