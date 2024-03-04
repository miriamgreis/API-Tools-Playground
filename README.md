# Playground for API Tools based on OpenAPI

This repository can be used to try out some tools for validation and contract-testing based on OpenAPI.
We use all tools via `npx`, so you only need `npm` to run the examples.
As sample API, we use the Swagger Petstore API.

## Validation with @stoplight/spectral-cli

[Spectral](https://github.com/stoplightio/spectral) is a tool that can be used to lint OpenAPI files and AsyncAPI files.

Spectral takes the OpenAPI file and validates it against a ruleset in a file called `.spectral.yaml`. 
You can use a recommended ruleset and write custom rules on top.

### Execution

Execute the command to run the validation in your terminal in the root folder of the repository:
```
npx @stoplight/spectral-cli lint openapi.yaml
```
`openapi.yaml` is the name of the OpenAPI file of our example in the repository.

Several validation errors will show up:
```
230:20  warning  operation-description       Operation "description" must be present and non-empty string.  paths./pet/{petId}.post.description
```
They always contain the line, the severity, a name of the triggered rule, the description of the validation error, and the path to the field in the OpenAPI definition.

## Validation with @quobix/vacuum

[Vacuum](https://quobix.com/vacuum/) is a tool, which can be used for bundling and linting OpenAPI files.

It is inspired by Spectral and compatible with custom Spectral rulesets.

### Execution

Execute the command to run the validation in your terminal in the root folder of the repository:
```
npx @quobix/vacuum lint --details openapi.yaml
```
`openapi.yaml` is the name of the OpenAPI file of our example in the repository.

Several validation errors will show up:
```
openapi.yaml:812:5  | warning  | `securitySchemes` component `api_key` is missing a description                      | component-description      | Descriptions | $.components.securitySchemes['api_key']
```
They always contain the line, the severity, a description of the validation error, the name of the rule, the category, and the path to the field in the OpenAPI definition.

### How to Fix the Errors

Start with one of the validation errors and try to identify the wrong field in the OpenAPI validation.
Fix the validation error and run the validation again.
In the best case, you shouldn't have any validation errors at the end.

## Contract Testing with @apideck/portman

For contract testing, we can use the tools [Portman](https://github.com/apideck-libraries/portman/tree/main) and [Newman](https://github.com/postmanlabs/newman).

Portman takes an OpenAPI file and generates a Postman collection out of it including tests.
With the standard configuration, contract tests are generated for each operation testing
- that the HTTP call was successful
- that the response has the correct content type, JSON body and schema
- if any specified headers are present

Further tests and configurations can be added by modifying the configuration file.

Newman is the command-line runner from Postman and runs the tests in the generated Postman collection.

### Execution

Execute the command to run the contract tests in your terminal in the root folder of the repository:
```
npx @apideck/portman --local openapi.yaml --portmanConfigFile portman-test-config.json --output postmanCollection.json
```

`openapi.yaml` is the name of the OpenAPI file, `portman-test-config.json` is the name of the configuration file for the tests, and `postmanCollection.json` is the name of the generated postman collection.
Run the contract test in your terminal in the same folder by passing the name of the generated postman collection:
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
        "value": "<your-api-key"
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
