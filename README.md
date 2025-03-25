# Playground for API Tools based on OpenAPI

This repository can be used to try out some tools for OpenAPI.
Right now it contains examples for mocking, validation, formatting and contract testing.
We use all tools via `npx`, so you only need `npm` to run the examples.
As sample API, we use the Swagger Petstore API.

Just copy the commands to your terminal in the root folder of the repository and get started.

## Mocking with @stoplight/prism-cli

[Prism](https://github.com/stoplightio/prism) is a tool that can be used to easily create a mock server from your OpenAPI files.
It additionally offers a validation proxy to add contract testing to existing test suites.

### Execution for Mocking

Execute the following command to start the mock server:
```
npx @stoplight/prism-cli mock openapi.yaml
```

In another terminal (or any other tool of your choice), you can now call the mock server:
```
curl http://127.0.0.1:4010/stations --header "Authorization: Bearer 1234"
```

## Validation with @stoplight/spectral-cli

[Spectral](https://github.com/stoplightio/spectral) is a tool that can be used to validate OpenAPI files and AsyncAPI files.

Spectral takes the OpenAPI file and validates it against a ruleset in a file called `.spectral.yaml`. 
You can use a recommended ruleset and write custom rules on top.

### Execution

Execute the following command to validate the OpenAPI definition:
```
npx @stoplight/spectral-cli lint openapi.yaml
```
`openapi.yaml` is the name of the OpenAPI file of our example in the repository.

Several validation errors will show up:
```
2:6   warning  info-contact   Info object must have "contact" object.      info
```
They always contain the line, the severity, a name of the triggered rule ([see Spectral ruleset](https://docs.stoplight.io/docs/spectral/4dec24461f3af-open-api-rules)), the description of the validation error, and the path to the field in the OpenAPI definition.

## Validation with @quobix/vacuum

[Vacuum](https://quobix.com/vacuum/) is a tool, which can be used for bundling and validating OpenAPI files.
It is inspired by Spectral and compatible with custom Spectral rulesets.

### Execution

Execute the following command to validate the OpenAPI definition:
```
npx @quobix/vacuum lint --details openapi.yaml
```
`openapi.yaml` is the name of the OpenAPI file of our example in the repository.

Several validation errors will show up:
```
openapi.yaml:990:5   | warning  | `schemas` component `Problem` is missing a description   | component-description  | Descriptions | $.components.schemas['Problem']                                                                   
```
They always contain the line, the severity, a description of the validation error, the name of the rule, the category, and the path to the field in the OpenAPI definition.

### How to Fix the Errors

Start with one of the validation errors and try to identify the wrong field in the OpenAPI validation.
Fix the validation error and run the validation again.
In the best case, you shouldn't have any validation errors at the end.

## Formatting with openapi-format

[openapi-format](https://github.com/thim81/openapi-format) is a tool to order, format and filter fields in OpenAPI files. 
This helps to create a more clean and optimizied OpenAPI file for public documentation.

### Execution

Execute the following command to sort your OpenAPI file by applying the default sort order of the tool:
```
npx openapi-format openapi.yaml --output openapi_formatted.yaml
```
Compare the two files to check for the changes.

### Filtering Example

Create a new file in the root folder of this repository called `customFilter.yaml`. 
Then, add the following content to the file:
```
flags:
  - x-internal
```

Open your OpenAPI file and place the `x-internal: true` flag on different elements. 
Then run:
```
npx openapi-format openapi.yaml --output openapi_formated.yaml --filterFile customFilter.yaml
```

The newly formatted file should not contain any of the flagged elements.

You can use many other filter options, and also pass other command line options such as `--sortFile` for a custom sort order or `--casingFile` to specify case settings. 
Check the documentation of the tool for more details.

## Contract Testing with @apideck/portman

For contract testing, we can use the tools [Portman](https://github.com/apideck-libraries/portman/tree/main) and [Newman](https://github.com/postmanlabs/newman).

Portman takes an OpenAPI file and generates a Postman collection out of it including tests.
With the standard configuration, contract tests are generated for each operation testing
- that the HTTP call was successful
- that the response has the correct content type, JSON body and schema
- if any specified headers are present

Further tests and configurations can be added by modifying the configuration file.

Newman is the command-line runner from Postman and runs the tests in the generated Postman collection.

### Execution for Validation

Execute the following command to generate the Postman collection including tests.
```
npx @apideck/portman --local openapi.yaml --portmanConfigFile portman-test-config.json --output postmanCollection.json
```

`openapi.yaml` is the name of the OpenAPI file, `portman-test-config.json` is the name of the configuration file for the tests, and `postmanCollection.json` is the name of the generated postman collection.
Run the contract test by passing the name of the generated postman collection:
```
npx newman run postmanCollection.json --verbose
```

I always recommend to use the `--verbose` option because I find it helpful to identify the problems if the tests don't pass.
If you want to, you can try it without later to compare and make your own decision.

Run the tests at least once and check the output before you continue.

### Order of Operations

When we execute the tests for the first time, many of them will fail.
One reason is the execution order of the contract tests.
Portman executes all operations depending on the tag order and their position in the OpenAPI file.
However, it makes more sense to create an entity first and then reuse the ID of the created entity to read, update, and at the end delete it.
Otherwise, we might try to delete an entity which doesn't even exist.

We can use the `orderOfOperations` keyword in Portman to change the execution order.
Open the `portman-test-config.json` and add the following configuration to the `globals` object at the bottom of the file:
```json
"orderOfOperations": [
    "POST::/*",
    "GET::/*",
    "PUT::/*",
    "DELETE::/*"
]
```

We can also extract the ID from the entity created in the contract test by replacing the empty `assignVariables` section with the following configuration:
```json
"assignVariables": [
    {
        "openApiOperation": "POST::/pet",
        "collectionVariables": [
            {
                "responseBodyProp": "id",
                "name": "petId"
            }
        ]
    }
]
```
The snippet takes the property `id` of the specified operation `POST::/pet` and stores it in a Postman variable called `petId`.

To pass this variable to other operations, we can use the `keyValueReplacements`.
We can replace the currently empty configuration with the following one:
```json
"keyValueReplacements": {
    "petId": "{{petId}}"
}
```
All keys with the name `petId` get the value of the Postman variable `petId`.

When we run the tests again now, we can check that the operation order is correctly applied
and the `petId` is passed on to the next operations.

### How to Fix the Errors

Pick one operation with failing tests and explore why the tests are failing.
Usually, it is easy to fix most of the contract test errors because we know the API which we are testing very well.
Knowing the expected behavior, we can quickly spot what's going wrong.
This is not the case for the Swagger Petstore API.
To better understand the errors, it is helpful to open the Swagger Petstore API in the [Swagger Editor](https://editor.swagger.io) and experiment with the tryout to understand the expected behavior of the operations.
Then try to correct the OpenAPI file to make the tests pass.

### Test Our Own API

When we test other APIs, we might face additional challenges regarding the specific URL to test or authorization.

#### Test Specific URL

For our sample API, the tests automatically use the correct URL for calling the API.
This might be different, for APIs containing multiple server URLs or when we want to test a specific environment not mentioned in the OpenAPI file.
We can easily pass the `baseUrl` parameter to Portman to specify the concrete URL to use for the tests:

```
npx @apideck/portman --local openapi.yaml --portmanConfigFile portman-test-config.json --baseUrl https://petstore3.swagger.io/api/v3 --output postmanCollection.json
```

#### Authorization

If the API which we want to test requires authorization, we can use the `securityOverwrites` in Portman. 
You can add the configuration to the `globals` object.
However, overwrites just work if the OpenAPI contains the respective security scheme which we want to overwite.

For an API key, you might use the following snippet:
```json
"securityOverwrites": {
    "apiKey": {
        "value": "<your-api-key>"
    }
}
```

For a Bearer token, you might use the following snippet:
```json
"securityOverwrites": {
    "bearer": {
        "token": "<your-bearer-token>"
    }
}
```

For testing locally now, we can add our credentials directly to the `portman-test-config.json`.
When we share the file or want to use Portman in a pipeline, we should use variables.
