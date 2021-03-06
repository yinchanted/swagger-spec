# Reuse Philosophy
We encourage reuse and patterns through references.

## What is reusable
The following types are reusable, as defined by the spec:

* Operations
* Parameters
* Responses
* Models (or Schema Objects in general)

## Reuse strategy
When reusing components in an API design, a pointer is created from the definition to target design.  The references are maintained in the structure, and can be updated by modifying the source definitions.  This is different from a "copy on design" approach where references are injected into the design itself.

The reuse technique is transparent between JSON or YAML and is lossless when converting between the two.

YAML anchors are technically allowed but break the general reuse strategy in Swagger, since anchors are "injected" into a single document.  They are not recommended.

Referenes can be made either inside the Swagger definition file or to external files. References are done using [JSON Reference](http://tools.ietf.org/html/draft-pbryan-zyp-json-ref-03).

## Techniques

### Guidelines for Referencing

When referencing internally, the target references have designated locations:

* Parameters -> `parameters`
* Responses -> `responses`
* Models (and general Schema Objects) -> `definitions`

Operations can only be referenced externally.

An example for an internal reference - `#/definitions/MyModel`. All references are canonical and must be a qualified [JSON Pointer](http://tools.ietf.org/html/rfc6901). For example, simply referencing `MyModel` is not allowed, even if there are no other definitions of it in the file.

When referencing externally, use a valid URI as the reference value. If your referenced file contains only one definition that should be used, you can refer to the file directly. For example:

_Assuming file https://my.company.com/definitions/Model.json_
```json
{
  "description": "A simple model",
  "type": "object",
  "properties": {
    "id": {
      "type": "integer"
    }
  }
}
```

The reference would be `https://my.company.com/definitions/Model.json`.

_Assuming file https://my.company.com/definitions/models.json_
```json
{
  "models": {
    "Model": {
      "description": "A simple model",
      "type": "object",
      "properties": {
        "id": {
          "type": "integer"
        }
    },
    "Tag": {
      "description": "A tag entity in the system",
      "type": "object",
      "properties": {
        "name": {
          "type": "string"
        }
      }
    }
  }
}
```

The reference to the `Model` model would be `https://my.company.com/definitions/models.json#/models/Model`. Make sure you include the full path to the model itself, including its containers if needed.

Whether you reference definitions internally or externally, you can never override or change their definitions from the referring location. The definitions can only be used as-is.

### Definitions
Reuse schema definitions by creating a repository of definitions.  This is done by simply hosting a file or set of files for commonly used definitions across a company or organization.  In the case of multiple files, the models can be referenced directly as such:

_Assuming file https://my.company.com/definitions/Model.json_
```json
{
  "description": "A simple model",
  "type": "object",
  "properties": {
    "id": {
      "type": "integer",
      "format": "int64"
    },
    "tag": {
      "description": "a complex, shared property.  Note the absolute reference",
      "$ref": "https://my.company.com/definitions/Tag.json"
    }
  }
}
```

For a single file, you can package the definitions in an object:

_Assuming file https://my.company.com/definitions/models.json_
```json
{
  "models": {
    "Model": {
      "description": "A simple model",
      "type": "object",
      "properties": {
        "id": {
          "type": "integer",
          "format": "int64"
        },
        "tag": {
          "description": "a complex, shared property.  Note the absolute reference",
          "$ref": "https://my.company.com/definitions/models.json#/models/Tag"
        }
      }
    },
    "Tag": {
      "description": "A tag entity in the system",
      "type": "object",
      "properties": {
        "name": {
          "type": "string"
        }
      }
    }
  }
}
```


### Parameters
Similar to model schemas, you can create a repository of `parameters` to describe the common entities that appear throughout a set of systems.  Using the same technique as above, you can host on either a single or multiple files.  For simplicity, the example below assumes a single file.

_Assuming file https://my.company.com/parameters/parameters.json_

```json
{
  "query" : {
    "skip": {
      "name": "skip",
      "in": "query",
      "description": "Results to skip when paginating through a result set",
      "required": false,
      "minimum": 0,
      "type": "integer",
      "format": "int32"
    },
    "limit": {
      "name": "limit",
      "in": "query",
      "description": "Maximum number of results to return",
      "required": false,
      "minimum": 0,
      "type": "integer",
      "format": "int32"
    }
  }
}
```

To include these parameters, you would need to add them individually as such:

```json
{
  "/pets": {
    "get": {
      "description": "Returns all pets from the system that the user has access to",
      "produces": [
        "application/json"
      ],
      "responses": {
        "200": {
          "description": "A list of pets.",
          "parameters" : [
            {
              "$ref": "https://my.company.com/parameters/parameters.json#/query/skip"
            },
            {
              "$ref": "https://my.company.com/parameters/parameters.json#/query/limit"
            },
            {
              "in": "query",
              "name": "type",
              "description": "the types of pet to return",
              "required": false,
              "type": "string"
            }
          ],
          "schema": {
            "type": "array",
            "items": {
              "$ref": "#/definitions/pet"
            }
          }
        }
      }
    }
  }
}
```

### Operations
Again, Operations can be shared across files.  Although the reusability of operations will be less than with Parameters and models. For this example, we will share a common `health` resource so that all APIs can reference it:

```json
{
  "/health": {
    "$ref": "http://localhost:8000/operations.json#/health"
  }
}
```

Which points to the reference in the `operations.json` file:

```json
{
  "health": {
    "get": {
      "tags": [
        "admin"
      ],
      "summary": "Returns server health information",
      "operationId": "getHealth",
      "produces": [
        "application/json"
      ],
      "parameters": [],
      "responses": {
        "200": {
          "description": "Health information from the server",
          "schema": {
            "$ref": "http://localhost:8000/models.json#/Health"
          }
        }
      }
    }
  }
}
```

Remember, you cannot override the definitions, but in this case, you can add additional operations on the same path level.

### Responses
Just like the other objects, responses can be reused as well.

Assume the file `responses.json`:

```json
{
  "NotFoundError": {
    "description": "Entity not found",
    "schema": {
      "$ref": "#/definitions/ErrorModel"
    }
  }
}
```

You can refer to it from a response definition:
```json
{
  "/pets/{petId}": {
    "get": {
      "tags": [
        "pet"
      ],
      "summary": "Returns server health information",
      "operationId": "getHealth",
      "produces": [
        "application/json"
      ],
      "parameters": [
        {
          "name": "petId",
          "in": "path",
          "description": "ID of pet to return",
          "required": true,
          "type": "integer",
          "format": "int64"
        }
      ],
      "responses": {
        "200": {
          "description": "The pet",
          "schema": {
            "$ref": "#/definitions/Pet"
          }
        },
        "400": {
          "$ref": "http://localhost:8000/responses.json#/NotFoundError"
        }
      }
    }
  }
}
```

### Constraints
* Referenced objects must be to JSON structures.  YAML reuse structures may be supported in a future version.
