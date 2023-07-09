# JSON schema validator for JSON for Modern C++

<!-- SHINX-START -->

This is a C++ library for validating JSON documents based on a
[JSON Schema](http://json-schema.org/) which itself should validate with
[draft-7 of JSON Schema Validation](http://json-schema.org/schema).

First a disclaimer: *It is work in progress and
contributions or hints or discussions are welcome.*

Niels Lohmann et al develop a great JSON parser for C++ called [JSON for Modern
C++](https://github.com/nlohmann/json). This validator is based on this
library, hence the name.

## Getting started

Currently, this package only offers a C++ library interface, and is only
available via cmake's `FetchContent` and conan. It is highly recommended
to use cmake to link to this library

Dependencies:

- NLohmann's Json library: At least **3.6.0**
  (See Github actions for officially tested versions)

### CMake configuration

<!-- SHINX-CMAKE-START -->

Bellow is a minimum cmake configuration file using `FetchContent`:

```cmake
cmake_minimum_required(VERSION 3.11)

project(example)

include(FetchContent)

FetchContent_Declare(nlohmann_json_schema_validator
        GIT_REPOSITORY pboettch/json-schema-validator
        # Please use a specific version tag
        GIT_TAG main
        )
FetchContent_MakeAvailable(nlohmann_json_schema_validator)

add_executable(example main.cpp)
target_link_libraries(example PRIVATE nlohmann_json_schema_validator::validator)
```

<!-- SHINX-CMAKE-END -->

For more details about the available cmake options and recommended configurations
see [docs/cmake](docs/cmake/index.md)

### Api example

See also `app/json-schema-validate.cpp`.

```C++
#include <iostream>
#include <iomanip>

#include <nlohmann/json-schema.hpp>

using nlohmann::json;
using nlohmann::json_schema::json_validator;

// The schema is defined based upon a string literal
static json person_schema = R"(
{
    "$schema": "http://json-schema.org/draft-07/schema#",
    "title": "A person",
    "properties": {
        "name": {
            "description": "Name",
            "type": "string"
        },
        "age": {
            "description": "Age of the person",
            "type": "number",
            "minimum": 2,
            "maximum": 200
        }
    },
    "required": [
                 "name",
                 "age"
                 ],
    "type": "object"
}

)"_json;

// The people are defined with brace initialization
static json bad_person = {{"age", 42}};
static json good_person = {{"name", "Albert"}, {"age", 42}};

int main()
{
    /* json-parse the schema */

    json_validator validator; // create validator

    try {
        validator.set_root_schema(person_schema); // insert root-schema
    } catch (const std::exception &e) {
        std::cerr << "Validation of schema failed, here is why: " << e.what() << "\n";
        return EXIT_FAILURE;
    }

    /* json-parse the people - API of 1.0.0, default throwing error handler */

    for (auto &person : {bad_person, good_person}) {
        std::cout << "About to validate this person:\n"
                  << std::setw(2) << person << std::endl;
        try {
            validator.validate(person); // validate the document - uses the default throwing error-handler
            std::cout << "Validation succeeded\n";
        } catch (const std::exception &e) {
            std::cerr << "Validation failed, here is why: " << e.what() << "\n";
        }
    }

    /* json-parse the people - with custom error handler */
    class custom_error_handler : public nlohmann::json_schema::basic_error_handler
    {
        void error(const nlohmann::json_pointer<nlohmann::basic_json<>> &pointer, const json &instance,
            const std::string &message) override
        {
            nlohmann::json_schema::basic_error_handler::error(pointer, instance, message);
            std::cerr << "ERROR: '" << pointer << "' - '" << instance << "': " << message << "\n";
        }
    };


    for (auto &person : {bad_person, good_person}) {
        std::cout << "About to validate this person:\n"
                  << std::setw(2) << person << std::endl;

        custom_error_handler err;
        validator.validate(person, err); // validate the document

        if (err)
            std::cerr << "Validation failed\n";
        else
            std::cout << "Validation succeeded\n";
    }

    return EXIT_SUCCESS;
}
```

## Compliance

There is an application which can be used for testing the validator with the
[JSON-Schema-Test-Suite](https://github.com/json-schema-org/JSON-Schema-Test-Suite).
In order to simplify the testing, the test-suite is included in the repository.

If you have cloned this repository providing a path the repository-root via the
cmake-variable `JSON_SCHEMA_TEST_SUITE_PATH` will enable the test-target(s).

All required tests are **OK**.

## Format

Optionally JSON-schema-validator can validate predefined or user-defined formats.
Therefore a format-checker-function can be provided by the user which is called by
the validator when a format-check is required (ie. the schema contains a format-field).

This is how the prototype looks like and how it can be passed to the validation-instance:

```C++
static void my_format_checker(const std::string &format, const std::string &value)
{
	if (format == "something") {
		if (!check_value_for_something(value))
			throw std::invalid_argument("value is not a good something");
	} else
		throw std::logic_error("Don't know how to validate " + format);
}

// when creating the validator

json_validator validator(nullptr, // or loader-callback
                         my_format_checker); // create validator
```

### Default Checker

The library contains a default-checker, which does some checks. It needs to be
provided manually to the constructor of the validator:

```C++
json_validator validator(loader, // or nullptr for no loader
                         nlohmann::json_schema::default_string_format_check);
```

Supported formats: `date-time, date, time, email, hostname, ipv4, ipv6, uuid, regex`

More formats can be added in `src/string-format-check.cpp`. Please contribute implementions for missing json schema draft formats.

### Default value processing

As a result of the validation, the library returns a json patch including the default values of the specified schema.

```C++
#include <iostream>
#include <nlohmann/json-schema.hpp>

using nlohmann::json;
using nlohmann::json_schema::json_validator;

static const json rectangle_schema = R"(
{
    "$schema": "http://json-schema.org/draft-07/schema#",
    "title": "A rectangle",
    "properties": {
        "width": {
            "$ref": "#/definitions/length",
            "default": 20
        },
        "height": {
            "$ref": "#/definitions/length"
        }
    },
    "definitions": {
        "length": {
            "type": "integer",
            "minimum": 1,
            "default": 10
        }
    }
})"_json;

int main()
{
	try {
		json_validator validator{rectangle_schema};
		/* validate empty json -> will be expanded by the default values defined in the schema */
		json rectangle = "{}"_json;
		const auto default_patch = validator.validate(rectangle);
		rectangle = rectangle.patch(default_patch);
		std::cout << rectangle.dump() << std::endl; // {"height":10,"width":20}
	} catch (const std::exception &e) {
		std::cerr << "Validation of schema failed: " << e.what() << "\n";
		return EXIT_FAILURE;
	}
	return EXIT_SUCCESS;
}
```

The example above will output the specified default values `{"height":10,"width":20}` to stdout.

> Note that the default value specified in a `$ref` may be overridden by the current instance location. Also note that this behavior will break draft-7, but it is compliant to newer drafts (e.g. `2019-09` or `2020-12`).

<!-- SHINX-END -->
