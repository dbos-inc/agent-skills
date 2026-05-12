---
title: Use Proper Test Fixtures for DBOS
impact: LOW-MEDIUM
impactDescription: Ensures clean state between tests
tags: testing, pytest, fixtures, reset
---

## Use Proper Test Fixtures for DBOS

Use pytest fixtures to properly reset DBOS state between tests.

**Incorrect (no reset between tests):**

```python
def test_workflow_one():
    DBOS.launch()
    result = my_workflow()
    assert result == "expected"

def test_workflow_two():
    # DBOS state from previous test!
    result = another_workflow()
```

**Correct (reset fixture):**

```python
import pytest
import os
from dbos import DBOS, DBOSConfig

@pytest.fixture()
def reset_dbos():
    DBOS.destroy()
    config: DBOSConfig = {
        "name": "test-app",
        "system_database_url": os.environ.get("TESTING_DATABASE_URL"),
    }
    DBOS(config=config)
    DBOS.reset_system_database()
    DBOS.launch()
    yield
    DBOS.destroy()

def test_workflow_one(reset_dbos):
    result = my_workflow()
    assert result == "expected"

def test_workflow_two(reset_dbos):
    # Clean DBOS state
    result = another_workflow()
    assert result == "other_expected"
```

The fixture:
1. Destroys any existing DBOS instance
2. Creates fresh configuration
3. Resets the system database
4. Launches DBOS
5. Yields for test execution
6. Cleans up after test

To minimize test dependencies, you can point `system_database_url` at SQLite instead of Postgres:

```python
config: DBOSConfig = {
    "name": "test-app",
    "system_database_url": "sqlite:///my_test_db.sqlite",
}
```

### Mocking Steps

Workflows and steps are ordinary Python functions, so you can mock them with `unittest.mock`:

```python
from unittest.mock import patch

def test_workflow(reset_dbos):
    with patch("myapp.main.get_data") as mock_get:
        mock_get.return_value = [...]
        with patch("myapp.main.record_data") as mock_record:
            mock_record.return_value = True
            my_workflow(input)
            mock_get.assert_called_once_with(expected_args)
```

Reference: [Testing DBOS](https://docs.dbos.dev/python/tutorials/testing)
