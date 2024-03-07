# Flask

Flask is for me one of the best frameworks for web development applications in Python, is well used abroad and is very flexible, leaving to the developers the role of building the project the way they want.

See the [documentation](https://flask.palletsprojects.com/).

The boilerplate for a starting project might be like the below, create a file called `app.py` (it does not need to be this name, it can be any name, unless `Flask.py`):

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello_world():
    return "<p>Hello, World!</p>"
```

Then run like the following, the parameter `--` app app `points to the starter file, for our case is` app.py `without the` .py and run is the command. Other commands might be added also and used in the same form.

```bash
flask --app app run
```

## My way of structuring the Flask application

Because Flask is very flexible, there is no specific way of structuring the project, it will depend on someone's feelings and wishes, my way of doing it is described below.

1. Create the `app.py` starter file
...

2. Use Blueprints for contained modules
...
Add a configuration file for the main stuff happening on Flask...

3. Create an `infra` folder for all abstractions on infrastructure
...

4. Use commands for the usual actions on the flask ...
Improvement is my middle name, so this list might change a lot, and it will be noticed on my GitHub projects.

## A way of transforming Flask into an app

There is a Python package called preview that encapsulates a "webview" into standalone windows running behind GUI engines such as PyQT. The applications are many, but for our case, we can pass the starter flask object to it and it will render for us, like below:

```python
import webview
from flask import Flask

app = Flask(__name__)

# Pretend the rest of the flask application is here

webview.create_window('Flask Application', app)
webview.start()
```

The `webview.start()` is the entrypoint of the application now, so instead of using Flask `run` you need only to run the python file as usual like below:

```bash
python app.py
```

> TODO: Create an example with commands in flask.

I created a list of Q&A sections with all the things I found out about this tool.

* _What Happens if _my application _has_ many pages_?_: All the pages of the Flask application will open normally on the contained window, unless they have a different hostname, like a link for another application/web page.

...
