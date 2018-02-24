# Modern C++ JSON schema validator

# What is it?

This is a C++ library for validating JSON documents based on a
[JSON Schema](http://json-schema.org/) which itself should validate with
[draft-4 of JSON Schema Validation](http://json-schema.org/schema).

First a disclaimer: *It is work in progress and
contributions or hints or discussions are welcome.*

Niels Lohmann et al develop a great JSON parser for C++ called [JSON for Modern
C++](https://github.com/nlohmann/json). This validator is based on this
library, hence the name.

The name is for the moment purely marketing, because there is, IMHO, not so much
modern C++ inside. There is plenty of space to make it more modern.

External documentation is missing as well. However the API of the validator
will be rather simple.

# Design goals

The main goal of this validator is to produce *human-comprehensible* error
messages if a JSON-document/instance does not comply with its schema. This is
done with exceptions thrown at the users with a helpful message telling what's
wrong with the document while validating.

Another goal was to use Niels Lohmann's JSON-library. This is why the validator
lives in his namespace.

# Weaknesses

Schema-reference resolution is not recursivity-proven: If there is a nested
cross-schema reference, it will not stop.  (Though I haven't tested it)

# How to use

## Build

Directly

```Bash
git clone https://github.com/pboettch/json-schema-validator.git
cd json-schema-validator
mkdir build
cd build
cmake .. \
    -DNLOHMANN_JSON_DIR=<path/to/json.hpp> \
    -DJSON_SCHEMA_TEST_SUITE_PATH=<path/to/JSON-Schema-test-suite> # optional
make # install
ctest # if test-suite has been given
```
or from another CMakeLists.txt as a subdirectory:

```CMake
# create an interface-target called json-hpp
add_library(json-hpp INTERFACE)
target_include_directories(json-hpp
    INTERFACE
        path/to/json.hpp)

# set this path to schema-test-suite to get tests compiled - optional
set(JSON_SCHEMA_TEST_SUITE_PATH "path/to/json-schema-test-suite")
enable_testing() # if you want to inherit tests

add_subdirectory(path-to-this-project json-schema-validator)
```

## Code

See also `app/json-schema-validate.cpp`.

```C++
#include <iostream>

#include "json-schema.hpp"

using nlohmann::json;
using nlohmann::json_uri;
using nlohmann::json_schema_draft4::json_validator;

// The schema is defined based upon a string literal
static json person_schema = R"(
{
    "$schema": "http://json-schema.org/draft-04/schema#",
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

int main(){
    
    /* json-parse the schema */
    
    json_validator validator; // create validator
    
    try {
        validator.set_root_schema(person_schema); // insert root-schema
    } catch (const std::exception &e) {
        std::cerr << "Validation of schema failed, here is why: " << e.what() << "\n";
        return EXIT_FAILURE;
    }
    
    /* json-parse the people */
    
    for (auto &person : {bad_person, good_person})
    {
        std::cout << "About to validate this person:\n" << std::setw(2) << person << std::endl;
        try {
            validator.validate(person); // validate the document
            std::cout << "Validation succeeded\n";
        } catch (const std::exception &e) {
            std::cerr << "Validation failed, here is why: " << e.what() << "\n";
        }
    }
    return EXIT_SUCCESS;
}

```

# Compliance

There is an application which can be used for testing the validator with the
[JSON-Schema-Test-Suite](https://github.com/json-schema-org/JSON-Schema-Test-Suite).

If you have cloned this repository providing a path the repository-root via the
cmake-variable `JSON_SCHEMA_TEST_SUITE_PATH` will enable the test-target(s).

All required tests are **OK**.

**12** optional tests of **305** total (required + optional) tests are failing:

- 10 of them are `format`-strings which are not supported.
- big numbers are not working (2)

# Additional features

## Default values

The goal is to create an empty document, based on schema-defined
default-values, recursively populated.

