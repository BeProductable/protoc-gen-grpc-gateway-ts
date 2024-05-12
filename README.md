# TypeScript Client Generator for gRPC Gateway

`protoc-gen-grpc-gateway-ts` is a TypeScript client generator for the [grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway/) project. It generates idiomatic TypeScript clients that connect the web frontend and golang backend fronted by grpc-gateway.

> [!NOTE]
> The official gRPC ecosystem repository hasn't been updated in since 2022 and there are a number of pending pull requests and issues that would be good to resolve. @seanami and @dpup work for two separate organizations using this fork and intend to breath a bit of life into this project. We hope these changes can be merged upstream at some point.

## Features:

1. Idiomatic Typescript clients and messages.
2. Supports both one way and server side streaming gRPC calls.
3. POJO request construction guarded by message type definitions, which is way easier compare to `grpc-web`.
4. No need to use swagger/open api to generate client code for the web.

### Changes made since the fork

1. [Support for well-known wrapper types](https://github.com/grpc-ecosystem/protoc-gen-grpc-gateway-ts/pull/50)
2. Updated to support gRPC gateway v2 and latest protoc-gen-go
3. Support for proto3 optional fields
4. Updated to satisfy strict TS and eslint checks
5. Generator options managed through standard flags
6. Fixes inconsistent field naming when fields contain numbers, e.g. `k8s_field` --> `k8sField`
7. Fixes module names when they contain dots or dashes

## Getting Started:

### Install `protoc-gen-grpc-gateway-ts`

You will need to install `protoc-gen-grpc-gateway-ts` before it could be picked up by the `protoc` command. Just run `go install github.com/dpup/protoc-gen-grpc-gateway-ts`

### Sample Usage:

`protoc-gen-grpc-gateway-ts` should be used along with the `protoc` command. A sample invocation looks like the following:

```bash
protoc --grpc-gateway-ts_out . \
  --grpc-gateway-ts_opt ts_import_roots=$(pwd) \
  --grpc-gateway-ts_opt ts_import_root_aliases=base \
  input.proto
```

As a result the generated file will be `input.pb.ts` in the same directory.

## Parameters:

### `use_proto_names`

To keep the same convention with `grpc-gateway` v2 & `protojson`. The field name in message generated by this library is in lowerCamelCase by default. If you prefer to make it stick the same with what is defined in the proto file, this option needs to be set to true.

### `emit_unpopulated`

To match `grpc-gateway` v2 & `protojson` with the `EmitUnpopulated` option. If true, zero values will be emited instead of dropped, so `{"str": ""}` instead of `{}`.

### `ts_import_roots`

Since protoc plugins do not get the import path information as what's specified in `protoc -I`, this parameter gives the plugin the same information to figure out where a specific type is coming from so that it can generate `import` statement at the top of the generated typescript file. Defaults to `$(pwd)`

### `ts_import_root_aliases`

If a project has setup an alias for their import. This parameter can be used to keep up with the project setup. It will print out alias instead of relative path in the import statement. Default to "".

`ts_import_roots` & `ts_import_root_aliases` are useful when you have setup import alias in your project with the project asset bundler, e.g. Webpack.

### `fetch_module_directory` and `fetch_module_filename`

`protoc-gen-grpc-gateway-ts` generates a shared typescript file with communication functions. These two parameters together will determine where the fetch module file is located. Default to `$(pwd)/fetch.pb.ts`

### `logtostderr`

Turn on logging to stderr. Default to false.

### `loglevel`

Defines the logging levels. Default to info. Valid values are: debug, info, warn, error

### Notes:

Zero-value fields are omitted from the URL query parameter list for GET requests. Therefore for a request payload such as `{ a: "A", b: "" c: 1, d: 0, e: false }` will become `/path/query?a=A&c=1`. A sample implementation is present within this [proto file](https://github.com/dpup/protoc-gen-grpc-gateway-ts/blob/master/test/integration/service.proto) in the integration tests folder. For further explanation please read the following:

- <https://developers.google.com/protocol-buffers/docs/proto3#default>
- <https://github.com/googleapis/googleapis/blob/master/google/api/http.proto>

## Examples:

The following shows how to use the generated TypeScript code.

Proto file: `counter.proto`

```proto
// file: counter.proto
message Request {
  int32 counter = 1;
}

message Response {
  int32 result = 1;
}

service CounterService {
  rpc Increase(Request) returns (Response);
  rpc Increase10X(Request) returns (stream Response);
}
```

Run the following command to generate the TypeScript client:

`protoc --grpc-gateway-ts_out=. counter.proto`

Then a `counter.pb.ts` file will be available at the current directory. You can use it like the following example.

```typescript
import { CounterService } from "./counter.pb";

// increase the given number once
async function increase(base: number): Promise<number> {
  const resp = await CounterService.Increase({ counter: base });
  return resp.result;
}

// increase the base repeatedly and return all results returned back from server
// the notifier after the request will be called once a result comes back from server streaming
async function increaseRepeatedly(base: number): Promise<number[]> {
  let results = [];
  await CounterService.Increase10X({ base }, (resp: Response) => {
    result.push(resp.result);
  });

  return results;
}
```

If you wish to make use of optional fields, you may find it beneficial to to configure the gRPC Gateway to emit zero values. Otherwise the client needs to know that a field is expected to be optional. See the `http get request with optional fields` test case to see the differences in response objects.

```go
  gateway := runtime.NewServeMux(runtime.WithMarshalerOption(runtime.MIMEWildcard, &runtime.HTTPBodyMarshaler{
    Marshaler: &runtime.JSONPb{
      MarshalOptions: protojson.MarshalOptions{
        EmitUnpopulated: true,
      },
    },
  }))
```

You should then pass `emit_unpopulated` as a parameter to the typescript plugin:

```bash
  protoc \
  ... \
  --grpc-gateway-ts_out ./ \
  --grpc-gateway-ts_opt emit_unpopulated=true \
  ...
```

## Development

Required dependencies:

```
brew install typescript
brew install golangci-lint
```

To run tests:

```
make testdata
make go-tests
make integration-tests
```

### Details on integration tests

The integration test has a number of steps:

1. `./scripts/gen-protos.sh` ensures updated typescript definitions.
2. `main.go` is executed and starts a server that loads up the protos.
3. The JS integration test in `integration_test.ts` is run using `Karma`.

The server side code is currently checked in to git. Changes need to be updated manually by running `./scripts/gen-server-protos.sh`.

## License

```text
Copyright 2020 Square, Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```
