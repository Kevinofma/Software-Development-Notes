# FastAPI and Pydantic Tutorial

FastAPI is a modern, fast (high-performance), standards-first web framework for Python. FastAPI helps you both specify and build RESTful HTTP APIs quickly.

Pydantic is a library used by FastAPI for data modeling and validation. It is how we will specify the schemas for request and response body data. It enforces type hints at runtime and yields user-friendly errors.

## Part 1. Starting a FastAPI and Pydantic Project

Before diving in, make sure you've reviewed [Getting Started](index.md). We are going to be following the same steps outlined there with a few tweaks to the dev container and requirements.

### Step 1. Setting up the Development Container

After following the initial steps on creating a new project, in the devcontainer, we want to use the following template:

```json
{
    "image": "mcr.microsoft.com/vscode/devcontainers/python:3.13",
    "customizations": {
        "vscode": {
            "extensions": [
                "ms-python.python",
                "ms-python.black-formatter"
            ],
            "settings": {
                "[python]": {
                    "editor.defaultFormatter": "ms-python.black-formatter"
                },
                "python.formatting.blackArgs": [
                    "--line-length",
                    "120"
                ],
                "editor.formatOnSave": true,
                "python.linting.enabled": true,
                "python.linting.pylintEnabled": true,
                "python.analysis.typeCheckingMode": "basic"
            }
        }
    },
    "postCreateCommand": "pip install -r requirements.txt"
}
```

??? note "Explanation of each field in this updated container"

    * **image**: Uses a specific Python 3.13 development container from Microsoft’s registry. This ensures your whole team is running the same Python version.  

    * **customizations**: Configures VS Code inside the container.  

        * **extensions**:  
            - `ms-python.python`: The official Python extension, adding linting, IntelliSense, debugging, etc.  
            - `ms-python.black-formatter`: Integrates the Black code formatter directly into VS Code.  

        * **settings**:  
            - `[python].editor.defaultFormatter`: Ensures Python files use Black for formatting.  
            - `python.formatting.blackArgs`: Sets Black’s line length limit to 120 characters (instead of the default 88).  
            - `editor.formatOnSave`: Automatically formats files on save, enforcing consistent style.  
            - `python.linting.enabled`: Enables linting so you catch errors as you type.  
            - `python.linting.pylintEnabled`: Uses Pylint as the linter.  
            - `python.analysis.typeCheckingMode`: Enables *basic* type checking (catches simple type mismatches).  

    * **postCreateCommand**: After the container is first built, this runs `pip install -r requirements.txt` so dependencies are available immediately.  


    ??? info "What is Black?"

        **Black** is a popular Python code formatter.  
        - It rewrites code to follow a consistent style (an "opinionated" formatter).  
        - This eliminates arguments about spaces, commas, or line breaks, making all code look uniform.  
        - The container sets Black’s line length to **120 characters** (instead of the default 88).  
        - It’s configured to run **automatically on save** in VS Code.

        Example:  

        ```python
        # Before Black
        def add(a,b): return a+b

        # After Black
        def add(a, b):
            return a + b
        ```

    ??? info "What is Pylint?"

        **Pylint** is a Python *linter* — a tool that analyzes your code for errors, bugs, and style issues.  
        - It checks for things like unused variables, undefined names, and poor practices.  
        - Pylint also enforces Python’s PEP 8 style guide, helping ensure readability.  
        - Unlike Black (which automatically changes code), Pylint only **reports issues** and gives you suggestions.  

        Example output:  

        ```text
        myscript.py:1:0: C0114: Missing module docstring (missing-module-docstring)
        myscript.py:3:4: C0103: Variable name "xYz" doesn't conform to snake_case naming style (invalid-name)
        ```

        In this container, **linting is enabled by default** so that mistakes are caught as you type.  

### Step 2. Updating `requirements.txt`

In addition to setting up the container, we need to update the `requirements.txt` file so our project installs the correct dependencies.

Add the following lines to your `requirements.txt` file at the root of your project:

```txt
fastapi[standard]~=0.115.7
black~=24.10.0
pylint~=3.3.4
```
  
??? note "**Explanation:**"
    
    * **fastapi[standard]~=0.115.7**  
      Installs FastAPI along with its *standard extras* (like `uvicorn` for running the server).  
      The `~=` means “compatible release” — so it allows patch-level updates (e.g., `0.115.8`), but not breaking upgrades.  

    * **black~=24.10.0**  
      Ensures that the **Black formatter** is available inside the container.  
      Even though the VS Code extension runs formatting, the CLI tool is needed for automation and consistency.  

    * **pylint~=3.3.4**  
      Installs **Pylint** so the linter runs inside the container.  
      This ensures that linting works consistently across environments, even outside VS Code.

---

## Part 2. Defining a Route

Create a main.py in the root directory of your project. This is where you will define your routes.
In FastAPI, routes define the *endpoints* of your API — the URLs that clients can send requests to. A route specifies both:

- **The HTTP method** (e.g., `GET`, `POST`, `PUT`, etc.)  
- **The path** (e.g., `/`, `/users`, `/items/42`)  

Here’s a simple example in `main.py`:

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root() -> str:
    return "Hello, world!"
```

??? info "Breaking down the example"

    * **`app = FastAPI()`**  
        Creates a FastAPI application instance. All routes are registered to this `app`.  

    * **`@app.get("/")`**  
        This is a **decorator** that registers the function below it as the handler for `GET` requests sent to the `/` path (the root URL).  
        * `get` → the HTTP method  
        * `"/"` → the path  
        Together, this means: *“When a client sends a GET request to `/`, run the following function.”*  

    * **`def read_root() -> str:`**  
        A standard Python function that is called when the route is hit.  
        * The function name (`read_root`) is arbitrary, but should describe what the route does.  
        * The return type annotation (`-> str`) is optional, but clarifies that this function returns a string.  

    * **`return "Hello, world!"`**  
        The function’s return value becomes the **HTTP response body**.  
        * If you return a string, FastAPI automatically wraps it into a valid HTTP response.  
        * For more complex data, you typically return a dictionary or Pydantic model, which FastAPI will automatically convert to JSON.  

!!! abstract "Key Idea"

    Every route in FastAPI follows this same pattern:

    ```python
    @app.<method>("<path>")
    def <function_name>(...):
        return <response>
    ```

    Where:

    * `<method>` is one of `get`, `post`, `put`, `patch`, or `delete`.
    * `<path>` is the URL endpoint.
    * `<function_name>` is any descriptive name you choose.
    * `<response>` is the data you want to send back to the client.

---

## Part 3. Running the Development Server

To run your FastAPI app in development, navigate to your project folder in the terminal and run:
```bash
fastapi run main.py --reload
```

??? info "What this does"
    * **Runs at http://127.0.0.1:8000**  
    By default, FastAPI’s dev server listens on this address and port.  
    *Note:* If another server (like your MkDocs dev server) is using port 8000, check the **Ports** tab in VS Code to see what host port this container’s 8000 is mapped to.  

    * **`--reload` argument**  
    Tells FastAPI to watch your source files. When changes are detected, the server automatically reloads.  
    This saves you from stopping and restarting the server each time you make edits.  

    * **Uvicorn**  
    Under the hood, FastAPI runs on **Uvicorn**, an ASGI server that handles low-level HTTP requests.  
    You usually don’t need to worry about it, but if you see references to Uvicorn in docs or logs, just know it’s the HTTP layer beneath FastAPI.  

Whenever a request is sent to `GET /`, FastAPI calls the `read_root()` function you defined.

---

## Part 4. Using Pydantic Models to Define Schemas

In FastAPI, **Pydantic models** let you define the structure of your data in Python. They serve two main purposes:

1. **Server-side Python objects** – You can use the models throughout your code as typed classes.

2. **API schema validation** – FastAPI automatically generates JSON schemas from your models for request and response validation.

### How to Define a Pydantic Model

1. Import `BaseModel` from `pydantic`.

2. Create a Python class that inherits from `BaseModel`.

3. Add attributes with type annotations to define the data structure.

Example of creating a BaseModel:
    ```python
    from pydantic import BaseModel

    class Item(BaseModel):
        id: int
        name: str
        description: str | None = None
    ```

??? info "Explanation"
    * `id` and `name` are required fields.

    * `description` is optional (None by default).

    * The class itself acts as both a Python object and a schema definition.

### Using Models in Routes

You can use Pydantic models for:

* **Request bodies** – FastAPI will validate incoming JSON and automatically convert it into a model instance.

* **Response bodies** – FastAPI will convert model instances to JSON for the client.

Example of returning a list of model instances:
    ```python
    from fastapi import FastAPI

    app = FastAPI()

    # Example "database"
    items_db: dict[int, Item] = {}

    @app.get("/items")
    def list_items() -> list[Item]:
        return list(items_db.values())
    ```
??? info "Explanation"
    * items_db stores Python objects that are instances of Item.

    * list_items returns a list of Pydantic models. FastAPI automatically converts these to JSON.

    * The function’s return type (list[Item]) specifies the expected response schema.

??? Example "Example All Together"
    ```python
    from fastapi import FastAPI
    from pydantic import BaseModel

    app = FastAPI()

    class Post(BaseModel):
        id: int
        content: str

    # Prepopulate dictionary of posts
    posts_db = {
        1: Post(id=1, content="Hello FastAPI!"),
        2: Post(id=2, content="Writing my second post!")
    }

    @app.get("/")
    def read_root() -> str:
        return "Hello, world!"

    @app.get("/about")
    def read_about() -> str:
        return "This is a simple HTTP API."

    @app.get("/posts")
    def list_posts() -> list[Post]:
        return list(posts_db.values())
    ```

    !!! info "How This Works"
        * We store two example posts in a global dictionary, `posts_db`, keyed by their ID.
        * The route `GET /posts` returns `list(posts_db.values())`, which effectively returns all posts as a list.
        * Notice how each value in `posts_db` is already an instance of `Post`. When FastAPI sees these objects, it converts them to JSON automatically.

        Notice the return type of the `list_posts` function is a `list` of `Post` objects. This is specifying the response body schema. Try visiting this route in your browser to confirm it is working. If you do not see well formatted JSON that is easy to read, try going back to the previous part of this reading and installing a JSON Viewer plugin in your web browser.

!!! tip "Key Points to Remember"

    * **Validation happens automatically** – If the data doesn’t match the model’s types, FastAPI returns a clear error response.

    * **Optional fields** can be specified with `field: type | None = default`.

    * **Nested models** are allowed – a model attribute can itself be another Pydantic model.

    * **Type hints** are crucial – FastAPI relies on them for validation, documentation, and auto-completion.

This approach ensures your API responses and requests are consistent, readable, and easy to validate. You can now expand your API by defining more models for different resources or more complex nested data structures.

---

## Part 5. Adding a Dynamic Route

So far, we’ve only worked with static paths like `/` or `/items`. But real APIs often need dynamic paths where part of the URL is a variable. For example:

* `/items/1` → fetch item with ID 1
* `/users/alice` → fetch data for the user `"alice"`

```python
In FastAPI, this is done using path parameters.

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    id: int
    name: str

# Pretend database
items_db = {
    1: Item(id=1, name="Widget"),
    2: Item(id=2, name="Gadget"),
}

@app.get("/items/{item_id}")
def get_item(item_id: int) -> Item:
    if item_id in items_db:
        return items_db[item_id]
    raise HTTPException(status_code=404, detail="Item not found")
```

### How it Works
* **`@app.get("/items/{item_id}")`**

    * The `{item_id}` part makes the path dynamic.

    * Whatever value is passed in the URL gets captured and passed into the function as the `item_id` parameter.

* **`item_id: int`**

    * Declaring `item_id` as an `int` means FastAPI automatically validates it.

    * If the client provides a non-integer (like `/items/abc`), FastAPI responds with a 422 Unprocessable Entity error.

* **Returning a model**

    * If the ID exists, the function returns the corresponding `Item` (a Pydantic model).

    * FastAPI automatically converts it into JSON.

* **Raising an exception**

    * If the ID is not found, the function raises an `HTTPException`.

    * This is how you return error responses in FastAPI.

??? example "Try It Out"

    ✅ **Happy path**: Visit `/items/1` or `/items/2`. You’ll get a valid JSON response with item data.

    ❌ **Not found**: Visit `/items/99`. You’ll get a 404 Not Found response, because the item isn’t in our database.

    ⚠️ **Invalid type**: Visit `/items/abc`. You’ll get a 422 Unprocessable Entity response, because `"abc"` is not an integer.

!!! tip "Why This Matters"

    Dynamic routes let you build APIs that interact with specific resources.

    * `/users/{user_id}` → fetch a specific user

    * `/orders/{order_id}` → fetch details for a given order

    * `/products/{sku}` → fetch a product by stock-keeping unit

    FastAPI handles **both validation and error reporting automatically**, reducing boilerplate code.   x

---

!!! info "Automatic API Documentation"

    One of FastAPI’s biggest benefits is that it automatically generates **interactive API documentation**.  

    * Visit **`/docs`** → Opens a web interface (built on OpenAPI) where you can:
        - See all your endpoints.
        - Try them out directly in your browser.
        - View sample requests and responses.  

    * Visit **`/openapi.json`** → Shows the raw JSON specification for your API, which can be used by tools or code generators.  

    Because FastAPI uses **Pydantic models** for request and response bodies, the documentation automatically includes:
    - Field types  
    - Validation rules  
    - Possible error states  

    ✅ This makes `/docs` your go-to place to **explore, test, and understand your API** while developing.

---

## Part 6. Adding a POST Route

Make your API writable by letting clients create new resources. Below is an example that uses the same `Item` model and `items_db` used earlier so it fits with the existing examples.

```python
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    id: int
    name: str
    description: str | None = None

# Example in-memory "database" (already used elsewhere in this tutorial)
items_db = {
    1: Item(id=1, name="Widget"),
    2: Item(id=2, name="Gadget"),
}

@app.post("/items", status_code=status.HTTP_201_CREATED)
def create_item(item: Item):
    if item.id in items_db:
        raise HTTPException(status_code=400, detail="Item with this ID already exists")
    items_db[item.id] = item
    return item
```

??? info "Request body & validation"
    When a client sends JSON to `POST /items`, FastAPI automatically:

    * Parses the request body into a `Item` instance (because the handler declares `item: Item`).

    * Runs validation using Pydantic (type checks, required fields, optional fields).

    If validation fails, FastAPI returns a **422 Unprocessable Entity** response with details about the problem.

??? note "Status code"
    The decorator parameter `status_code=status.HTTP_201_CREATED` tells FastAPI to return a **201 Created** status on success.
    This is the conventional response for resource creation. If you instead want to replace an existing resource, use `PUT` (idempotent) or return a different status code as appropriate.

??? example "Try it in `/docs`"
    1. Open `/docs`, find **POST /items** and click **Try it out**.  
    2. Provide a sample JSON body, for example:
    ```json
    {
      "id": 3,
      "name": "New Widget",
      "description": "A shiny new widget"
    }
    ```
    3. Click **Execute** and inspect the response (should be 201 and return the created item).  
    4. Verify by calling **GET /items/{item_id}** (or visiting `/items/3`) to see the new resource.

??? warning "In-memory storage is ephemeral"
    This example uses a Python dictionary (`items_db`) stored in module memory. It is **not persistent**:

    * When the server restarts or reloads, `items_db` resets to its initial contents.

    * For real applications, use a persistent database (Postgres, SQLite, etc.) so data survives restarts.

??? tip "Handling duplicates / alternatives"

    * This example returns **400 Bad Request** if an `id` already exists. Another pattern is:

        * Use `PUT /items/{id}` to create-or-replace a resource (idempotent).

        * Use server-assigned IDs (client posts without `id`) and return the new ID in the response.

## Part 7. Updating and Deleting Items

To complete our CRUD operations (Create, Retrieve, Update, Delete), we need to add routes for updating an existing item and deleting an item.  

Here’s an example using the same `Item` model and `items_db` dictionary introduced earlier:

```python
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    id: int
    name: str
    description: str | None = None

items_db = {
    1: Item(id=1, name="Widget"),
    2: Item(id=2, name="Gadget"),
}

@app.put("/items/{item_id}")
def update_item(item_id: int, updated_item: Item) -> Item:
    if item_id not in items_db:
        raise HTTPException(status_code=404, detail="Item not found")
    items_db[item_id] = updated_item
    return updated_item

@app.delete("/items/{item_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_item(item_id: int) -> None:
    if item_id not in items_db:
        raise HTTPException(status_code=404, detail="Item not found")
    del items_db[item_id]
    return None  # 204 = No Content
```

??? note "PUT: Update a Resource"
    * `PUT /items/{item_id}` replaces the resource at that URL with the new data sent in the request body.  
    * If the item doesn’t exist, return **404 Not Found**.  
    * The response includes the updated resource, so the client can confirm the change.  

??? note "DELETE: Remove a Resource"
    * `DELETE /items/{item_id}` removes the resource with that ID from the dictionary.  
    * A successful delete returns **204 No Content**, which means the action succeeded but there is nothing to return.  
    * If the item doesn’t exist, return **404 Not Found**.  

??? example "Trying it in `/docs`"
    With your dev server running:

    1. Go to `/docs` and expand the **PUT /items/{item_id}** section.  

        Enter an existing ID (e.g., `1`) and provide a body like:  

        ```json
        {
            "id": 1,
            "name": "Updated Widget",
            "description": "Improved version"
        }
        ```

        Execute, and check the response to see the updated item.  

    2. Confirm with **GET /items/{item_id}** that the item was updated.  

    3. Expand **DELETE /items/{item_id}** and provide the same ID.  
       Execute, and you’ll get a **204 response**.  
       Use **GET /items/{item_id}** again to confirm it was deleted (should return 404).  

??? tip "CRUD recap"
    With `POST`, `GET`, `PUT`, and `DELETE`, you now have the complete set of core HTTP methods for managing resources in your FastAPI app.
