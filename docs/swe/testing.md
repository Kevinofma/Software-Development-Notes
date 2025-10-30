# Introduction to Testing

## Why is Testing Important?

Software can be fragile; a small change in one place can unexpectedly break something else. Testing provides a safety net, ensuring that your code behaves as expected, bug fixes are permanent, and new features don't reintroduce old problems.

A good test suite doesn't just find bugs—it boosts your confidence to make changes. When you have reliable tests, you can refactor code and add new features without the fear of breaking existing functionality.

???+ note "Key Benefits of Testing"
    * **Verification**: Tests confirm that your software meets the intended requirements.
    * **Preventing Regressions**: Automated tests ensure that new code doesn't break existing features.
    * **Confidence in Changes**: Good test coverage makes you more confident when modifying or adding to your codebase.

---

## Types of Software Tests

Not all tests are the same. A balanced test suite will include different types of tests, each with a specific purpose. Being intentional about which tests you write is key to building a robust and maintainable application.

Here are some of the most common types of tests:

* **Unit Tests**: These focus on testing individual functions or methods in isolation. They are fast and great for verifying specific pieces of logic.
* **Integration Tests**: These tests check that different parts of your system work together correctly, such as your API interacting with a database.
* **End-to-End (E2E) Tests**: E2E tests simulate a full user workflow, from the user interface to the database and back. They provide high confidence that your entire system is working as expected.
* **Performance Tests**: These measure how your system performs under load, helping to identify bottlenecks and other performance issues.

For this guide, we'll focus on **unit and integration tests**, as they offer a great balance of speed, reliability, and value.

---

## Integration Testing a FastAPI Backend

Integration tests are perfect for verifying that your FastAPI application's components—from routing and dependency injection to your service layer—all work together.

### Setting up with `pytest`

We'll use `pytest`, a popular Python testing framework. To get started, create a `test_main.py` file. `pytest` will automatically discover and run tests in files named `test_*.py` or `*_test.py`.

Here's a basic integration test for our Rock, Paper, Scissors API:

!!! example "test_main.py"
    ```python
    from fastapi.testclient import TestClient
    from main import app

    client = TestClient(app)

    def test_play_route_integration():
        """Test that the /play endpoint handles basic gameplay correctly."""
        response = client.post("/play", json={"user_choice": "rock"})

        # Check the HTTP response
        assert response.status_code == 200
        assert response.headers["content-type"] == "application/json"

        # Check the response body
        data = response.json()
        assert "user_choice" in data
        assert data["user_choice"] == "rock"
        assert "api_choice" in data
        assert "user_wins" in data
        assert isinstance(data["user_wins"], bool)
    ```

??? question "What is `client = TestClient(app)`?"
    The `TestClient` is a special object provided by FastAPI (via Starlette) that allows you to make HTTP requests to your application *directly in your tests*, without needing to run a live web server.

    - **Why we need it**: It simulates a real client (like a web browser or a mobile app) sending requests to your API. This lets you test the entire request-response cycle, including routing, dependency injection, and response formatting, all within your test suite.
    - **How it works**: You initialize it with your FastAPI `app` instance. Then, you can use methods like `client.post()`, `client.get()`, etc., to interact with your endpoints just as a real client would.

???+ success "Example Input and Output"
    * **Input**: The test sends a `POST` request to the `/play` endpoint. The request body is a JSON object:
        ```json
        {"user_choice": "rock"}
        ```
    * **Output (Example)**: The `response` object's `.json()` method would return a dictionary similar to this (the `api_choice` and `timestamp` will vary):
        ```json
        {
          "timestamp": "2023-10-27T10:30:00.123456",
          "user_choice": "rock",
          "api_choice": "paper",
          "user_wins": false
        }
        ```
    The test then `assert`s that this output has the correct structure and data types.

---

## Unit Testing

While integration tests are great for the big picture, unit tests let us zoom in on individual components.

### Unit Testing the `GameService`

To test our `GameService`, we need a way to control the random choice it makes. We can use Python's `unittest.mock` to "patch" the `_random_choice` method and make it predictable.

!!! example "test_services.py"
    ```python
    from unittest.mock import MagicMock
    from services import GameService
    from models import Choice, GamePlay

    def create_mock_game_service(choice_to_return: Choice) -> GameService:
        """Create a GameService with a mocked _random_choice method."""
        service = GameService()
        service._random_choice = MagicMock(return_value=choice_to_return)
        return service

    def test_game_service_rock_beats_scissors():
        # Force the service to choose "scissors"
        service = create_mock_game_service(Choice.scissors)
        result = service.play(GamePlay(user_choice=Choice.rock))

        assert result.user_wins is True
        service._random_choice.assert_called_once()
    ```

### Unit Testing a Route Function

We can also unit test our route functions in isolation. To do this, we'll create a mock `GameService` and pass it to the route function directly.

!!! example "test_main_unit.py"
    ```python
    from unittest.mock import MagicMock
    from main import play
    from models import GamePlay, Choice

    def test_play_route_unit():
        # Create a mock service
        mock_service = MagicMock()
        user_choice = GamePlay(user_choice=Choice.rock)

        # Call the route function directly with the mock service
        play(user_choice=user_choice, game_svc=mock_service)

        # Verify that the route called the service's play method correctly
        mock_service.play.assert_called_once_with(user_choice)
    ```

??? info "What is `MagicMock`?"
    **`MagicMock`** is a powerful tool from Python's `unittest.mock` library that creates flexible "fake" objects for testing. Think of it as a stunt double for your real objects.

    - **What it does**: It creates an object that can pretend to be any other object. It automatically creates any methods or attributes you try to access on it.
    - **Why it's useful**: In our unit test, we want to test the `play` route function *without* using a real `GameService`. `MagicMock` lets us create a fake `game_svc` that we can control completely.
    - **Key Feature**: `MagicMock` records how it was used. After calling our route function with the mock, we can use assertion methods like `assert_called_once_with()` to verify that our route interacted with the service exactly as we expected. This proves that the route function correctly forwards its arguments to the service layer.

???+ success "Example Input and Output"
    This test doesn't check HTTP requests or responses, but rather the *interaction* between the route function and the service.

    * **Input**:
        1.  A `GamePlay` object: `GamePlay(user_choice=Choice.rock)`
        2.  A `MagicMock` object that stands in for our `GameService`.
    * **Action**: The test calls the `play` function directly, passing in the inputs.
    * **Output (Verification)**: The test doesn't return a value. Instead, it checks a side-effect: *Was the `play` method on our mock service called exactly once, and was it called with the correct `GamePlay` object?* The assertion `mock_service.play.assert_called_once_with(user_choice)` confirms this. If the route function had failed to call the service, or called it with the wrong data, the test would fail.

## The Limitations of Testing

It's important to remember that testing provides confidence, not a guarantee of bug-free software. It's impossible to test every possible input and scenario. A good test suite is a pragmatic one that focuses on the most critical and complex parts of your application.