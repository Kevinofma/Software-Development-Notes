# Service Layer and Dependency Injection

## Layered Architecture: Separating Concerns

One of the most foundational patterns in software engineering is the **layered architecture**. This pattern organizes a system into distinct layers, where each layer only depends on the one directly below it. This creates a clean separation of concerns and helps manage complexity, especially in larger applications.

A key rule is that **lower layers should not be aware of higher layers**. For example, a database layer shouldn't know about the API layer that uses it. In our FastAPI applications, we'll introduce a **business logic services layer** that sits between our API routes and our data persistence layer.

### What is the "Business Logic" Layer?

This layer is often called the **business logic** or **service layer**. It contains the core rules and workflows of your application. This logic is specific to your application's domain and is kept separate from technical details like how HTTP requests are handled or how data is stored.

The **API routing layer** acts as a translator, taking incoming HTTP requests, calling the appropriate methods in the service layer, and formatting the results into an HTTP response. The routes themselves should contain very little logic.

??? note "Why Separate Concerns?"

    Mixing all your application logic directly inside API route handlers can lead to several problems:

    * **Tightly Coupled Code**: The code becomes hard to modify because business logic and HTTP concerns are tangled together.
    * **Lack of Reusability**: You can't easily reuse your business logic in other places, like a command-line tool or a background worker.
    * **Difficult to Test**: It's challenging to test your business rules in isolation from the web framework.

By creating a separate service layer, we make our code more maintainable, reusable, and easier to test.

### Example of Layered Architecture

Let's look at a simple example of fetching a user.

!!! example "Without a Service Layer (Bad Practice)"

    ```python
    from fastapi import FastAPI, HTTPException
    from models import User

    app = FastAPI()

    @app.get("/users/{user_id}")
    def get_user(user_id: int):
        user = database.find_user_by_id(user_id)
        if not user:
            # Mixing HTTP-specific errors with business logic
            raise HTTPException(status_code=404, detail="User not found")
        return user

    In this case, the route handler knows about the database and also about `HTTPException`, which is specific to FastAPI.
    ```

!!! example "With a Service Layer (Good Practice)"
    Here, the responsibilities are split.
    **`services.py`**
    ```python
    from models import User

    class UserDoesNotExistError(Exception):
        pass

    class UserService:
        def get_user(self, user_id: int) -> User:
            user = database.find_user_by_id(user_id)
            if not user:
                # Raises a domain-specific error, not an HTTP one
                raise UserDoesNotExistError("User does not exist")
            return user
    ```
    The service layer contains no FastAPI code. It raises a custom error that is specific to the application's domain.

    **`main.py`**
    ```python
    from fastapi import FastAPI, HTTPException
    from services import UserService, UserDoesNotExistError
    from models import User

    app = FastAPI()
    user_service = UserService()

    @app.get("/users/{user_id}")
    def get_user(user_id: int):
        try:
            return user_service.get_user(user_id)
        except UserDoesNotExistError:
            # The route layer translates the domain error into an HTTP error
            raise HTTPException(status_code=404, detail="User not found")
    ```
    The route's job is to call the service and handle any domain-specific errors, translating them into appropriate HTTP responses.

---

## What is Dependency Injection?

**Dependency Injection (DI)** is a design pattern that is core to many modern frameworks like FastAPI, Spring, and Angular. The main idea is that instead of creating dependencies *inside* a function or class, you declare them as parameters. The framework then "injects" the required objects when it calls your function.

You've already been using DI in FastAPI\! When you define path parameters, query parameters, or request bodies in your route functions, FastAPI automatically provides (injects) those values for you. Now, we'll use it to inject our custom services.

### Why Use Dependency Injection?

DI makes your application:

  - **More Maintainable**: Components are loosely coupled, so changes in one part are less likely to break another.
  - **Easier to Test**: You can easily substitute dependencies with mock objects for unit testing.
  - **More Flexible**: You can swap out different implementations of a service without changing the code that uses it.
  - **Reduces Code Duplication**: Centralizes how dependencies are created and managed.

---

## Tutorial: Dependency Injection in FastAPI

Let's build a simple Rock, Paper, Scissors API to see DI in action.

!!! abstract "Goal"
    We will create a `GameService` to handle the game's logic and use FastAPI's `Depends` to inject it into our API routes.

### Step 1: Define the Pydantic Models

Create a file named `models.py` for our data structures.
```python
from enum import Enum
from datetime import datetime
from typing import Annotated, TypeAlias
from pydantic import BaseModel, Field

class Choice(str, Enum):
    rock = "rock"
    paper = "paper"
    scissors = "scissors"

ChoiceField: TypeAlias = Annotated[
    Choice,
    Field(
        description="Choice of rock, paper, or scissors.",
        examples=["rock", "paper", "scissors"],
    ),
]

class GamePlay(BaseModel):
    user_choice: ChoiceField

class GameResult(BaseModel):
    timestamp: Annotated[datetime, Field(description="When the game was played.")]
    user_choice: ChoiceField
    api_choice: ChoiceField
    user_wins: Annotated[bool, Field(description="Did the user win the game?")]
```

??? tip "Using `TypeAlias`"
    When you find yourself repeating complex type annotations, defining a `TypeAlias` is a great way to reduce boilerplate and improve readability.

### Step 2: Create the Game Service

Next, create a `services.py` file. This service will contain all the business logic for the game and will have no knowledge of FastAPI.
```python
from datetime import datetime
from random import choice as random_choice
from models import GamePlay, GameResult, Choice

# This is a simple in-memory store for game history.
# It's not a real database and will reset on server restart.
_db: list[GameResult] = []

class GameService:
    """Service for processing game plays."""

    def play(self, gameplay: GamePlay) -> GameResult:
        """Play a game round."""
        api_choice: Choice = self._random_choice()
        result = GameResult(
            timestamp=datetime.now(),
            user_choice=gameplay.user_choice,
            api_choice=api_choice,
            user_wins=self._does_user_win(gameplay.user_choice, api_choice),
        )
        _db.append(result)
        return result

    def get_results(self) -> list[GameResult]:
        """Get all game results."""
        return _db

    def _random_choice(self) -> Choice:
        """Select a random choice for the API."""
        return random_choice(list(Choice))

    def _does_user_win(self, user_choice: Choice, api_choice: Choice) -> bool:
        """Determine if the user wins."""
        result: tuple[Choice, Choice] = (user_choice, api_choice)
        winning_results: set[tuple[Choice, Choice]] = {
            (Choice.rock, Choice.scissors),
            (Choice.paper, Choice.rock),
            (Choice.scissors, Choice.paper),
        }
        return result in winning_results
```

### Step 3: Create the FastAPI Routes with Dependency Injection

Now, let's update `main.py` to use our `GameService`.

First, let's define a `TypeAlias` for our injected service to keep the route definitions clean.
```python
from typing import Annotated, TypeAlias
from fastapi import FastAPI, Body, Depends
from models import GamePlay, GameResult
from services import GameService

app = FastAPI()

# Create a TypeAlias for our dependency
GameServiceDI: TypeAlias = Annotated[GameService, Depends()]
```
Now, we can create the routes that *depend* on `GameService`.

```python
@app.post("/play")
def play(
    user_choice: Annotated[
        GamePlay,
        Body(description="User's choice of rock, paper, or scissors."),
    ],
    game_svc: GameServiceDI,  # FastAPI injects the service here!
) -> GameResult:
    return game_svc.play(user_choice)


@app.get("/results")
def get_results(game_svc: GameServiceDI) -> list[GameResult]:
    return game_svc.get_results()
```

!!! success "How It Works"
    * We added a `game_svc: GameServiceDI` parameter to our route functions.
    * The `Depends()` inside our `TypeAlias` signals to FastAPI that this is a dependency.
    * Before calling the route function, FastAPI will automatically create an instance of `GameService` and pass it in as the `game_svc` argument.
    * Our route handler is now clean and simple: its only job is to call the service and return the result. The creation of the service is handled by the framework.

This is the power of dependency injection\! Your route is no longer responsible for creating its dependencies, making it more focused and much easier to test. You could now easily write a unit test for the `play` function and provide a *mock* `GameService` to test its behavior in isolation.