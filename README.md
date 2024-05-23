# Golang ProtoBuf Validator Compiler

[![Travis Build](https://travis-ci.org/mwitkow/go-proto-validators.svg)](https://travis-ci.org/mwitkow/go-proto-validators)
[![Apache 2.0 License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

A `protoc` plugin that generates `Validate() error` functions on Go proto `struct`s based on field options inside `.proto`
files. The validation functions are code-generated and thus don't suffer on performance from tag-based reflection on
deeply-nested messages.

## Paint me a code picture

Let's take the following `proto3` snippet:

```proto
syntax = "proto3";
package validator.examples;
import "validator/validator.proto";

message InnerMessage {
  // some_integer can only be in range (0, 100).
  int32 some_integer = 1 [(validator.field) = {int_gt: 0, int_lt: 100}];
  // some_float can only be in range (0;1).
  double some_float = 2 [(validator.field) = {float_gte: 0, float_lte: 1}];
}

message OuterMessage {
  // important_string must be a lowercase alpha-numeric of 5 to 30 characters (RE2 syntax).
  string important_string = 1 [(validator.field) = {regex: "^[a-z0-9]{5,30}$"}];
  // proto3 doesn't have `required`, the `msg_exist` enforces presence of InnerMessage.
  InnerMessage inner = 2 [(validator.field) = {msg_exists : true}];
}
```

First, the **`required` keyword is back** for `proto3`, under the guise of `msg_exists`. The painful `if-nil` checks are taken care of!

Second, the expected values in fields are now part of the contract `.proto` file. No more hunting down conditions in code!

Third, the generated code is understandable and has clear understandable error messages. Take a look:

```go
func (x *InnerMessage) Validate() error {
	if !(x.SomeInteger > 0) {
		return fmt.Errorf("validation error: InnerMessage.SomeInteger must be greater than '0'")
	}
	if !(x.SomeInteger < 100) {
		return fmt.Errorf("validation error: InnerMessage.SomeInteger must be less than '100'")
	}
	if !(x.SomeFloat >= 0) {
		return fmt.Errorf("validation error: InnerMessage.SomeFloat must be greater than or equal to '0'")
	}
	if !(x.SomeFloat <= 1) {
		return fmt.Errorf("validation error: InnerMessage.SomeFloat must be less than or equal to '1'")
	}
	return nil
}

var _regex_OuterMessage_ImportantString = regexp.MustCompile("^[a-z0-9]{5,30}$")

func (x *OuterMessage) Validate() error {
	if !_regex_OuterMessage_ImportantString.MatchString(x.ImportantString) {
		return fmt.Errorf("validation error: OuterMessage.ImportantString must conform to regex '^[a-z0-9]{5,30}$'")
	}
	if nil == x.Inner {
		return fmt.Errorf("validation error: OuterMessage.Inner message must exist")
	}
	if x.Inner != nil {
		if err := validators.CallValidatorIfExists(this.Inner); err != nil {
			return err
		}
	}
	return nil
}
```

## Installing and using

The `protoc` compiler expects to find plugins named `proto-gen-XYZ` on the execution `$PATH`. So first:

```sh
export PATH=${PATH}:${GOPATH}/bin
```

Then, do the usual

```sh
go get github.com/liov/go-proto-validator/cmd/protoc-gen-validator
```

Your `protoc` builds probably look very simple like:

```sh
protoc  \
  --proto_path=. \
  --go_out=. \
  --validator_out=. \
  *.proto
```


Basically the magical incantation (apart from includes) is the `--validator_out`. That triggers the 
`protoc-gen-govalidator` plugin to generate `mymessage.validator.pb.go`. That's it :)

## License

`go-proto-validator` is released under the Apache 2.0 license. See the [LICENSE](LICENSE) file for details.
