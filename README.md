# Flask Pydantic Spec

A library to make it easy to add OpenAPI documentation to your Flask app, and validate the requests using [Pydantic](https://github.com/samuelcolvin/pydantic/).

This library began as a fork of [Spectree](https://github.com/0b01001001/spectree), but as we made changes we thought 
other people might be interested in our approach.

## Features

* Less boilerplate code, only annotations, no need for YAML :sparkles:
* Generate API document with [Redoc UI](https://github.com/Redocly/redoc) or [Swagger UI](https://github.com/swagger-api/swagger-ui) :yum:
* Validate query, JSON data, response data with [pydantic](https://github.com/samuelcolvin/pydantic/) :wink:
* Has support for request/response types other than JSON.

## Quick Start

install with pip: `pip install flask-pydantic-spec`

### Examples

Check the [examples](/examples) folder.

* [flask example](/examples/flask_demo.py)


### Step by Step

1. Define your data structure used in (query, json, headers, cookies, resp) with `pydantic.BaseModel`
2. create `flask_pydantic_spec.Validator` instance with the web framework name you are using, like `api = Validator('flask')`
3. `api.validate` decorate the route with
   * `query`
   * `body`
   * `headers`
   * `cookies`
   * `resp`
   * `tags`
4. access this data with `context(query, body, headers, cookies)` (of course, you can access these from the original place where the framework offered)
   * flask: `request.context`
5. register to the web application `api.register(app)`
6. check the document at URL location `/apidoc/redoc` or `/apidoc/swagger`

If the request doesn't pass the validation, it will return a 422 with JSON error message(ctx, loc, msg, type).

## How To

> How to add summary and description to endpoints?

Just add docs to the endpoint function. The 1st line is the summary, and the rest is the description for this endpoint.

> How to add description to parameters?

Check the [pydantic](https://pydantic-docs.helpmanual.io/usage/schema/) docs about description in `Field`.

> Any config I can change?

Of course. Check the [config](https://flask-pydantic-spec.readthedocs.io/en/latest/config.html) document.

You can update the config when you init the validator like: 

```py
from flask_pydantic_spec import FlaskPydanticSpec
FlaskPydanticSpec("flask", title="Demo API", version="v1.0", path="doc")
```

> What is a `Response` and how to use it?

To build a response for the endpoint, you need to declare the status code with format `HTTP_{code}` and corresponding data (optional).

```py
from flask_pydantic_spec import Response
Response(HTTP_200=None, HTTP_403=ForbidModel)
Response('HTTP_200') # equals to Response(HTTP_200=None)
```

> What should I return when I'm using the library?

No need to change anything. Just return what the framework required.

> How to logging when the validation failed?

Validation errors are logged with INFO level. Details are passed into `extra`.

> How can I change the response when there is a validation error? Can I record some metrics?

This library provides `before` and `after` hooks to do these. Check the [doc](https://flask-pydantic-spec.readthedocs.io/en/latest) or the [test case](tests/test_plugin_flask.py). You can change the handlers for Flask-Pydantic-Spec or for a specific endpoint validation.

## Demo

Try it with `http post :8000/api/user name=alice age=18`. (if you are using `httpie`)

### Flask

```py
from flask import Flask, request, jsonify
from pydantic import BaseModel, Field, constr
from flask_pydantic_spec import FlaskPydanticSpec, Response, Request


class Profile(BaseModel):
    name: constr(min_length=2, max_length=40) # Constrained Str
    age: int = Field(
        ...,
        gt=0,
        lt=150,
        description='user age(Human)'
    )

    class Config:
        schema_extra = {
            # provide an example
            'example': {
                'name': 'very_important_user',
                'age': 42,
            }
        }


class Message(BaseModel):
    text: str


app = Flask(__name__)
api = FlaskPydanticSpec('flask')


@app.route('/api/user', methods=['POST'])
@api.validate(body=Request(Profile), resp=Response(HTTP_200=Message, HTTP_403=None), tags=['api'])
def user_profile():
    """
    verify user profile (summary of this endpoint)

    user's name, user's age, ... (long description)
    """
    print(request.context.json) # or `request.json`
    return jsonify(text='it works')


if __name__ == "__main__":
    api.register(app) # if you don't register in api init step
    app.run(port=8000)

```

## FAQ

> ValidationError: missing field for headers

The HTTP headers' keys in Flask are capitalized.
You can use [`pydantic.root_validators(pre=True)`](https://pydantic-docs.helpmanual.io/usage/validators/#root-validators) to change all the keys into lower cases or upper cases.

> ValidationError: value is not a valid list for query

Since there is no standard way for HTTP query with multiple values, it's hard to find the way to handle this for different web frameworks. So I suggest not to use list type in query until I find a suitable way to fix it.
