# Parameter Validation with FastAPI

FastAPI leverages **Pydantic** and standard Python type hints to automatically perform data validation. While simple type hints (like `int`, `str`, or `float`) are sufficient for basic validation and conversion, you often need to enforce **additional constraints** on your URL parameters.

FastAPI provides the `Path` and `Query` functions to declare extra metadata and apply detailed validation rules to path and query parameters, respectively.

---

## 1. Dynamic Path Validation (Numeric)

Path parameters are required by definition, as they are part of the URL itself. You typically use path validation to ensure a parameter is a valid number within a specific range.

To add validation to a path parameter, you must import and use the **`Path`** function, often inside Python's **`Annotated`** type hint.

### Applying Numeric Constraints

You can use the following arguments within `Path()` to set up numeric validation:

| Argument | Description | Example Constraint |
|---|---|---|
| `gt` | **G**reater **T**han | `gt=10` (must be > 10) |
| `ge` | **G**reater than or **E**qual | `ge=10` (must be ≥ 10) |
| `lt` | **L**ess **T**han | `lt=100` (must be < 100) |
| `le` | **L**ess than or **E**qual | `le=100` (must be ≤ 100) |

```python
from typing import Annotated
from fastapi import FastAPI, Path

app = FastAPI()

@app.get("/items/{item_id}")
def read_items(
    # item_id must be an integer, greater than 0, and less than or equal to 100
    item_id: Annotated[int, Path(title="The ID of the item", gt=0, le=100)]
):
    """
    Retrieves an item by its ID, enforcing numeric constraints.
    If item_id is not a number, or outside the range 1-100, a 422 error is returned.
    """
    return {"item_id": item_id}
```

!!! note "Automatic Error Handling"

    If the client sends a value that violates these constraints (e.g., `/items/101` when `le=100`), FastAPI automatically returns a **422 Unprocessable Entity** error with clear details about which constraint was broken.

---

## 2. Query Parameter Validation (String)

Query parameters are defined as function parameters that are **not** part of the path. They are typically optional and are used for filtering or sorting.

To add validation to a query parameter, you must import and use the **`Query`** function, often within `Annotated`.

### Applying String Constraints

You can use the following arguments within `Query()` to set up string validation:

| Argument | Description | Example Constraint |
|---|---|---|
| `min_length` | Minimum required string length | `min_length=3` |
| `max_length` | Maximum allowed string length | `max_length=50` |
| `regex` | Regular expression pattern the string must match | `regex="^search.*"` |
| `alias` | Specify an alternative name for the parameter in the URL | `alias="s-query"` |

```python
from typing import Annotated, Union
from fastapi import FastAPI, Query

app = FastAPI()

@app.get("/search/")
def search_items(
    # q is optional (Union[str, None]), has a maximum length of 50, and must match a pattern
    q: Annotated[Union[str, None], Query(max_length=50, regex="^search.*", alias="s-query")] = None,
    # limit must be an integer, greater than 0, and is optional (default=25)
    limit: Annotated[int, Query(gt=0)] = 25
):
    """
    Search endpoint with validation on the query string (s-query) and limit.
    """
    results = {"q": q, "limit": limit}
    return results
```
!!! info "Understanding str | None = None"
    This common syntax is how you define an optional parameter that also has a type constraint:

    1.  **Type Hint (`str | None`):** This uses the modern **Union Operator (`|`)** in Python 3.10+. It declares that the variable `q` can hold either a **string** value (if the client provides it) or the special value **`None`** (if the client omits it).
    2.  **Default Value (`= None`):** Setting the default to `None` explicitly tells **FastAPI** that this parameter is **optional**. If you did not set a default value, FastAPI would treat `q` as required, even if the type hint included `| None`.

??? example "How `regex` and `alias` work together"

    When you define a parameter using both an `alias` and a `regex`, the following happens:

    1.  **`alias="s-query"`**: This tells FastAPI that the **URL parameter** a client sends must be named `s-query` (e.g., `GET /search/?s-query=search_term`) instead of using the function's internal name (`q`).
    2.  **`regex="^search.*"`**: This is a powerful validation rule. It ensures that the value provided for `s-query` **must start with the exact string `search`**.
        * ✅ **Valid:** `s-query=search_term`
        * ❌ **Invalid:** `s-query=find_term`

    ### Code Example

    In this function signature, `q` is the internal Python variable, but the client must use `s-query`:

    ```python
    from typing import Annotated, Union
    from fastapi import FastAPI, Query

    @app.get("/search/")
    def search_items(
        # The internal variable name is 'q'
        q: Annotated[Union[str, None], Query(alias="s-query", regex="^search.*")] = None
    ):
        # ... logic
        return {"query": q}
    ```
    
    ### Client Interaction

    | URL Request | Result |
    | :--- | :--- |
    | `/search/?s-query=search_widgets` | Success (200 OK) |
    | `/search/?s-query=find_widgets` | Failure (422 Unprocessable Entity) |
    | `/search/?q=search_widgets` | Success, but 'q' is treated as a separate, unvalidated parameter (because the alias was set). |

### Required vs. Optional Query Parameters

The required status of a query parameter is determined by its **default value** in the function signature:

  * **Optional:** If you set the default value to `None` or provide any default value (e.g., `limit=25`), the parameter is optional.
  * **Required:** If you do **not** set a default value, the parameter is required. To enforce requirements while still using `Query` for validation/metadata, you can use the literal **`...`** (Ellipsis) as the default value.

!!! tip "Make a Parameter Required"
    If you want a query parameter to be required without providing a specific default value, use `...` (Ellipsis) as the default.

    Example:
    ```python
    from fastapi import Query
    # ...
    # This query parameter is REQUIRED, has min_length 5, and no default value
    required_id: str = Query(min_length=5, default=...)
    ```

This use of `Path` and `Query` ensures that parameter validation is automatically handled and documented in your interactive **`/docs`** interface.