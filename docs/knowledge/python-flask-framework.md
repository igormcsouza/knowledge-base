---
tags:

- python
- flask
- web-development
- framework

---

# Flask Framework

Flask is one of the best frameworks for web development applications in Python. It's widely used and very flexible, giving developers the freedom to build projects the way they want.

See the [official documentation](https://flask.palletsprojects.com/).

## Basic Setup

The boilerplate for a starting project looks like this. Create a file called `app.py` (it doesn't need to be this name, it can be any name except `Flask.py`):

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello_world():
    return "<p>Hello, World!</p>"
```

Then run it with the following command. The parameter `--app app` points to the starter file (in our case `app.py` without the `.py` extension):

```bash
flask --app app run
```

## My Way of Structuring Flask Applications

Because Flask is very flexible, there's no specific way of structuring projects. It depends on personal preferences. Here's my approach:

### 1. Create the `app.py` Starter File

This serves as the entry point for your Flask application.

### 2. Use Blueprints for Contained Modules

Blueprints help organize your application into logical components and modules.

Add a configuration file for the main Flask functionality...

### 3. Create an `infra` Folder for Infrastructure Abstractions

This folder contains all infrastructure-related code and abstractions.

### 4. Use Commands for Common Flask Actions

Implement Flask CLI commands for routine operations and tasks.

!!! note
    This structure continues to evolve and improve. You can see updates in my GitHub projects.

## Converting Flask to a Desktop Application

There's a Python package called `webview` that encapsulates a web application into standalone windows using GUI engines like PyQt. Here's how to use it with Flask:

```python
import webview
from flask import Flask

app = Flask(__name__)

# The rest of your Flask application goes here

webview.create_window('Flask Application', app)
webview.start()
```

The `webview.start()` becomes the application's entry point. Instead of using Flask's `run` command, you simply run the Python file:

```bash
python app.py
```

### FAQ: Desktop Flask Applications

**What happens if my application has many pages?**
All pages of the Flask application will open normally in the contained window, unless they have a different hostname (like links to external applications/web pages).

!!! todo
    Create an example demonstrating Flask commands usage.

## Additional Resources

- [Flask Documentation](https://flask.palletsprojects.com/)
- [Flask Blueprints](https://flask.palletsprojects.com/en/2.3.x/blueprints/)
- [PyWebView](https://pywebview.flowrl.com/)
