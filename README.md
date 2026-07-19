# Flaskr — A Beginner's Guide to This Project

This project is a small blog application called **Flaskr**. It's based on the
official Flask tutorial, and it's a great "first project" because it touches
almost every core Flask concept without being overwhelming: routing,
templates, forms, sessions, a database, and authentication.

This document explains **what every file does, why it's written the way it
is, and how the pieces connect**. It's written for someone who is new to
Flask — so it favors clear explanations over brevity.

---

## 1. What the app actually does

Flaskr is a tiny blog:

- Anyone can **view** the list of blog posts (newest first).
- A visitor can **register** an account and **log in**.
- Once logged in, a user can **create** posts.
- A user can **edit** or **delete** only the posts *they* created.

That's it. Small feature set, but it needs a database, user accounts,
password security, and page templates — which is exactly why it's a good
teaching example.

---

## 2. The folder structure

```
flaskprojectlayout/
├── flaskr/                  ← the actual Python package (the "app")
│   ├── __init__.py          ← the "application factory" — builds the app
│   ├── db.py                ← database connection + setup helpers
│   ├── schema.sql           ← SQL that defines the database tables
│   ├── auth.py               ← registration / login / logout (a Blueprint)
│   ├── blog.py               ← viewing / creating / editing posts (a Blueprint)
│   ├── static/
│   │   └── style.css         ← CSS, served as-is to the browser
│   └── templates/            ← HTML files (Jinja2 templates)
│       ├── base.html         ← the shared page "skeleton"
│       ├── auth/
│       │   ├── login.html
│       │   └── register.html
│       └── blog/
│           ├── index.html
│           ├── create.html
│           └── update.html
├── instance/
│   └── flaskr.sqlite         ← the actual SQLite database file (created at runtime)
├── pyproject.toml            ← project metadata + dependencies (modern format)
└── requirements.txt          ← dependency pins (older/simpler format)
```

**Why is the code in a folder called `flaskr/` instead of one big `app.py`?**
Because this is a Python *package*, not a single script. Splitting the app
into a package means you can organize code by feature (`auth.py`, `blog.py`),
avoid one giant file, and — importantly — let Flask's tooling (`flask run`,
`flask init-db`) find and import it by name (`flaskr`).

---

## 3. `flaskr/__init__.py` — the Application Factory

This file is the entry point. When Python treats `flaskr/` as a package,
`__init__.py` is the code that runs when you `import flaskr`.

```python
def create_app(test_config=None):
    app = Flask(__name__, instance_relative_config=True)
    app.config.from_mapping(
        SECRET_KEY='dev',
        DATABASE=os.path.join(app.instance_path, 'flaskr.sqlite'),
    )
    ...
    return app
```

### Why a *function* that builds the app, instead of `app = Flask(__name__)` at the top of the file?

This pattern is called the **Application Factory pattern**, and it's one of
the most important ideas to take away from this project.

- **Testing**: Tests need a *fresh, isolated* app — often pointed at a
  temporary test database instead of the real one. If `app` were a global
  variable created at import time, every test would share the exact same
  app and database. With a factory, each test just calls
  `create_app(test_config)` and gets its own app instance.
- **Multiple configurations**: You might want a "dev" app and a
  "production" app with different settings (different `SECRET_KEY`,
  different `DATABASE` path, etc.). A factory lets you build either one on
  demand by passing in different config.
- **Avoiding import-time side effects**: Nothing "happens" just because you
  imported the `flaskr` package — the app is only built when someone
  actually calls `create_app()`. This avoids circular-import headaches,
  which is why you'll notice imports like `from . import db` happen
  **inside** the function body, not at the top of the file.

### Walking through the function line by line

```python
app = Flask(__name__, instance_relative_config=True)
```
- `__name__` tells Flask what package it belongs to, so Flask knows where to
  look for templates/static files relative to.
- `instance_relative_config=True` tells Flask that config files (and things
  like the SQLite database) should live in an `instance/` folder **outside**
  the `flaskr/` package. This is intentional — the `instance/` folder is
  meant to hold things that change at runtime (a real database file, secret
  config) and generally should **not** be committed to git or shipped as
  part of your source code.

```python
app.config.from_mapping(
    SECRET_KEY='dev',
    DATABASE=os.path.join(app.instance_path, 'flaskr.sqlite'),
)
```
- `SECRET_KEY` is used by Flask to cryptographically sign session cookies
  (see the sessions explanation in the `auth.py` section below). `'dev'` is
  a placeholder — fine for local development, but in a real deployed app
  this must be a long random secret, kept out of source control.
- `DATABASE` is just a config value (a file path) that `db.py` will read
  later to know where the SQLite file lives.

```python
if test_config is None:
    app.config.from_pyfile('config.py', silent=True)
else:
    app.config.from_mapping(test_config)
```
- If no test config was passed in, try to load `instance/config.py` (if it
  exists) to override defaults — this is where you'd put a real
  `SECRET_KEY` for production, for example. `silent=True` means "don't
  error if the file doesn't exist," which is expected in dev.
- If a `test_config` dict *was* passed in (this happens during automated
  tests), use that instead — so tests never touch your real config file.

```python
os.makedirs(app.instance_path, exist_ok=True)
```
- Makes sure the `instance/` folder actually exists on disk before anything
  tries to write the SQLite file into it.

```python
@app.route('/hello')
def hello():
    return 'Hello, World!'
```
- A minimal example route defined directly on the app, left over from the
  tutorial's "hello world" step. It's harmless, but in a real project you'd
  likely remove this once you understand the pattern — real routes live in
  the blueprints below.

```python
from . import db
db.init_app(app)

from . import auth
app.register_blueprint(auth.bp)

from . import blog
app.register_blueprint(blog.bp)
app.add_url_rule('/', endpoint='index')
```
- `db.init_app(app)` wires up database lifecycle hooks (explained in
  section 4) and adds the `flask init-db` CLI command.
- `register_blueprint(...)` is how the routes defined in `auth.py` and
  `blog.py` actually get attached to the running app (explained in section
  5 — **Blueprints**).
- `app.add_url_rule('/', endpoint='index')` makes the site's root URL `/`
  point at the `blog.index` view, so that `url_for('index')` works as a
  shorthand anywhere in the templates.

---

## 4. `flaskr/db.py` — Database connection management

SQLite is used here because it's just a single file on disk (`flaskr.sqlite`)
— no separate database server to install, perfect for learning.

```python
def get_db():
    if 'db' not in g:
        g.db = sqlite3.connect(
            current_app.config['DATABASE'],
            detect_types=sqlite3.PARSE_DECLTYPES
        )
        g.db.row_factory = sqlite3.Row
    return g.db
```

### What is `g`?

`g` (from `flask import g`) is a special object Flask gives you that is
**unique to a single request**. Anything you store on `g` during a request
is thrown away once that request finishes. It exists specifically so
different pieces of code handling the *same* request can share data without
passing it around as function arguments.

Here, `get_db()` uses `g` to cache the database connection: the **first**
time something calls `get_db()` during a request, it opens a new SQLite
connection and stores it as `g.db`. If anything else calls `get_db()` again
*during that same request*, it just reuses the same connection instead of
opening a new one. This avoids opening multiple redundant connections per
request.

- `current_app` is how you access the active Flask app instance from inside
  a function that isn't `create_app()` itself. It's necessary here because
  `db.py` needs to read `app.config['DATABASE']`, but `db.py` never
  received the `app` object directly — that's the trade-off of the factory
  pattern, and Flask's `current_app` proxy solves it.
- `sqlite3.Row` makes query results behave like dictionaries (e.g.
  `user['username']`) instead of plain tuples (`user[0]`), which is much
  more readable in the templates and views.

```python
def close_db(e=None):
    db = g.pop('db', None)
    if db is not None:
        db.close()
```
This closes the connection if one was opened. It's registered with:
```python
app.teardown_appcontext(close_db)
```
in `init_app()`. `teardown_appcontext` tells Flask "run this automatically
after every request finishes (even if it errored)." This guarantees the
database connection never leaks — you never have to remember to manually
close it in every view function.

```python
def init_db():
    db = get_db()
    with current_app.open_resource('schema.sql') as f:
        db.executescript(f.read().decode('utf8'))
```
This wipes and recreates the tables by running `schema.sql` (see section 6).
`open_resource` opens a file relative to the `flaskr` package, so this works
no matter what directory you run the command from.

```python
@click.command('init-db')
def init_db_command():
    """Clear the existing data and create new tables."""
    init_db()
    click.echo('Initialized the database.')
```
This registers a brand-new terminal command. Because it's added via
`app.cli.add_command(init_db_command)`, once your app is set up you can run:

```
flask --app flaskr init-db
```

...from the terminal to (re)create the database tables. This is how you set
up the database the very first time.

```python
sqlite3.register_converter(
    "timestamp", lambda v: datetime.fromisoformat(v.decode())
)
```
SQLite stores dates as plain text. This line tells Python's `sqlite3` module
"whenever a column declared as `TIMESTAMP` comes back from a query, convert
it into a real Python `datetime` object automatically" (this is what
`detect_types=sqlite3.PARSE_DECLTYPES` in `get_db()` enables). That's why
`post['created'].strftime(...)` works directly in
[blog/index.html](flaskr/templates/blog/index.html) — `created` is already a
real `datetime`, not a string.

---

## 5. Blueprints — organizing routes by feature

Both `auth.py` and `blog.py` start with:

```python
bp = Blueprint('auth', __name__, url_prefix='/auth')
```
```python
bp = Blueprint('blog', __name__)
```

### Why Blueprints instead of just adding all routes to `app` directly?

A **Blueprint** is a way to group a set of related routes, templates, and
logic together, and register them onto the app *later*, as a unit. Benefits:

- **Organization**: All authentication logic lives in one file
  (`auth.py`), all blog logic in another (`blog.py`), instead of one huge
  file with every route in it.
- **URL prefixing**: `auth`'s blueprint has `url_prefix='/auth'`, so a route
  defined as `@bp.route('/login')` inside it is automatically served at
  `/auth/login` — you don't repeat `/auth` everywhere.
- **Named endpoints**: Because the blueprint is named `'auth'`, its routes
  are referred to elsewhere as `auth.login`, `auth.register`, etc. (see
  `url_for('auth.login')` in the templates and views). This avoids name
  collisions — `blog.py` could theoretically have its own `index` name
  without conflicting with anything in `auth.py`.
- **Reusability/testability**: Blueprints can be registered, tested, or even
  omitted independently of the rest of the app.

The blueprint object itself does nothing until it's attached in
`create_app()` via `app.register_blueprint(auth.bp)`.

---

## 6. `flaskr/schema.sql` — the database structure

```sql
CREATE TABLE user (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  username TEXT UNIQUE NOT NULL,
  password TEXT NOT NULL
);

CREATE TABLE post (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  author_id INTEGER NOT NULL,
  created TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  title TEXT NOT NULL,
  body TEXT NOT NULL,
  FOREIGN KEY (author_id) REFERENCES user (id)
);
```

Two tables:

- **`user`**: `username` is `UNIQUE` so two people can't register the same
  name (SQLite enforces this and raises an `IntegrityError` if you try —
  see `auth.py`'s `register()`). Note the column is called `password`, but
  it never stores a plaintext password — only a *hash* of it (explained
  below).
- **`post`**: `author_id` is a **foreign key** pointing back to `user.id` —
  this is how the app knows who wrote each post. `created` automatically
  defaults to the current time when a row is inserted, which is why
  `blog.py`'s `INSERT` statement never needs to set it manually.

---

## 7. `flaskr/auth.py` — registration, login, logout

### Register

```python
@bp.route('/register', methods=('GET', 'POST'))
def register():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        ...
```

- A single view handles **both** showing the form (`GET`) and processing the
  submitted form (`POST`) — a very common Flask pattern. On `GET`, it falls
  through to `return render_template('auth/register.html')` at the bottom.
  On `POST`, it validates and processes the submitted data.
- `request.form['username']` reads a field from the submitted HTML form
  (see the `<input name="username">` in
  [register.html](flaskr/templates/auth/register.html)).

```python
db.execute(
    "INSERT INTO user (username, password) VALUES (?, ?)",
    (username, generate_password_hash(password)),
)
```

Two important security details here:

1. **`?` placeholders instead of string-formatting the SQL.** This is
   *parameterized querying*, and it's the standard defense against **SQL
   injection**. Never build a SQL string like
   `f"INSERT INTO user VALUES ('{username}')"` — a malicious username could
   contain SQL that breaks out of the string and manipulates your database.
   The `?` placeholders let the `sqlite3` driver safely handle escaping.
2. **`generate_password_hash(password)`** — the raw password is *never*
   stored. Instead, Werkzeug's `generate_password_hash` runs it through a
   one-way hashing algorithm. Even if your database were stolen, an
   attacker wouldn't have anyone's actual password. Login later uses
   `check_password_hash(user['password'], password)` to verify a submitted
   password matches the stored hash, without ever needing to "un-hash" it
   (hashing is intentionally one-directional).

```python
except db.IntegrityError:
    error = f"User {username} is already registered."
```
This catches the database error raised when the `UNIQUE` constraint on
`username` is violated — cleaner than manually querying "does this username
already exist?" before inserting.

### Login

```python
user = db.execute(
    'SELECT * FROM user WHERE username = ?', (username,)
).fetchone()

if user is None:
    error = 'Incorrect username.'
elif not check_password_hash(user['password'], password):
    error = 'Incorrect password.'

if error is None:
    session.clear()
    session['user_id'] = user['id']
    return redirect(url_for('index'))
```

### What is `session`?

`session` is a dictionary-like object tied to the visitor's browser via a
cookie. Flask automatically **cryptographically signs** this cookie using
`SECRET_KEY` (from `__init__.py`), so the browser can't tamper with its
contents (e.g. change `user_id` to someone else's id) — if the signature
doesn't match, Flask rejects the cookie. This is the whole mechanism behind
"staying logged in" between page loads — the browser sends the cookie back
on every request, and the server reads `session['user_id']` from it.

`session.clear()` before setting `user_id` wipes out any old session data
first (defensive — prevents leftover data from a previous user on a shared
browser, for instance).

### Keeping track of who's logged in on *every* request

```python
@bp.before_app_request
def load_logged_in_user():
    user_id = session.get('user_id')
    if user_id is None:
        g.user = None
    else:
        g.user = get_db().execute(
            'SELECT * FROM user WHERE id = ?', (user_id,)
        ).fetchone()
```

`before_app_request` registers a function that Flask runs **before every
single request**, across the whole app (not just the `auth` blueprint) —
that's what the `_app_` in the name means. Its job: look at the session
cookie, and if there's a logged-in `user_id`, load that user's full row from
the database and stash it on `g.user`. This is why every template can check
`{% if g.user %}` (see `base.html`) — by the time any view or template runs,
`g.user` has already been populated for you.

### Logout

```python
@bp.route('/logout')
def logout():
    session.clear()
    return redirect(url_for('index'))
```
Simply wipes the session cookie's contents — there's nothing server-side to
"log out" since Flask sessions are entirely client-side (just signed, not
stored in a database).

### Protecting routes: `login_required`

```python
def login_required(view):
    @functools.wraps(view)
    def wrapped_view(**kwargs):
        if g.user is None:
            return redirect(url_for('auth.login'))
        return view(**kwargs)
    return wrapped_view
```

This is a **decorator** — a function that wraps another function to add
behavior. `login_required` is used like this in `blog.py`:

```python
@bp.route('/create', methods=('GET', 'POST'))
@login_required
def create():
    ...
```

When a request comes in for `/create`, Flask actually calls
`wrapped_view()`, not `create()` directly. `wrapped_view` checks
`g.user` — if nobody's logged in, it redirects to the login page instead of
ever running the real `create()` code. This is a clean, reusable way to
gate any number of routes behind "must be logged in," without repeating the
same `if` check in every single view function.

`@functools.wraps(view)` is a small but important detail: it copies the
original function's name/metadata onto the wrapper, so Flask (and
debugging tools) still see it as `create` rather than a generic
`wrapped_view` — without it, having two decorated views both named
`wrapped_view` would confuse Flask's routing.

---

## 8. `flaskr/blog.py` — viewing, creating, editing, deleting posts

### Listing posts

```python
@bp.route('/')
def index():
    db = get_db()
    posts = db.execute(
        'SELECT p.id, title, body, created, author_id, username'
        ' FROM post p JOIN user u ON p.author_id = u.id'
        ' ORDER BY created DESC'
    ).fetchall()
    return render_template('blog/index.html', posts=posts)
```
This is registered as `blog.index`, but `__init__.py` also mapped the site's
`/` URL to it via `app.add_url_rule('/', endpoint='index')` — so it's the
homepage. It `JOIN`s `post` with `user` so it can show *who* wrote each post
(`username`) alongside the post itself, sorted newest-first.

### Creating a post

```python
@bp.route('/create', methods=('GET', 'POST'))
@login_required
def create():
    ...
    db.execute(
        'INSERT INTO post (title, body, author_id)'
        ' VALUES (?, ?, ?)',
        (title, body, g.user['id'])
    )
    db.commit()
```
Same GET/POST pattern as `register()`. Notice `db.commit()` — SQLite (via
Python's `sqlite3` module) requires you to explicitly commit a transaction
after any change (`INSERT`/`UPDATE`/`DELETE`), or the change won't actually
be saved to disk. `author_id` comes from `g.user['id']` — the currently
logged-in user, populated earlier by `load_logged_in_user()`.

### Fetching a single post safely — `get_post()`

```python
def get_post(id, check_author=True):
    post = get_db().execute(
        'SELECT p.id, title, body, created, author_id, username'
        ' FROM post p JOIN user u ON p.author_id = u.id'
        ' WHERE p.id = ?',
        (id,)
    ).fetchone()

    if post is None:
        abort(404, f"Post id {id} doesn't exist.")

    if check_author and post['author_id'] != g.user['id']:
        abort(403)

    return post
```
This is a shared helper (**not** a route itself — no `@bp.route` — it's a
regular function called *by* the `update` and `delete` views). It handles
two important edge cases in one place:

- **404 Not Found** if someone requests a post id that doesn't exist (e.g.
  visits `/999/update` for a post that was deleted).
- **403 Forbidden** if a logged-in user tries to edit/delete *someone else's*
  post. This is the authorization check that stops User A from tampering
  with User B's posts just by guessing a URL.

`check_author=True` is a default parameter, meaning callers can opt out of
the ownership check (not currently used that way here, but it shows the
function was designed to be reusable — e.g. a future "public view a single
post" page could call `get_post(id, check_author=False)`).

### Update and Delete

```python
@bp.route('/<int:id>/update', methods=('GET', 'POST'))
@login_required
def update(id):
    post = get_post(id)
    ...
```
`<int:id>` in the route is a **URL converter** — Flask extracts the number
from the URL (e.g. `/5/update`) and passes it into the view function as the
`id` argument, already converted to an `int`.

`delete` doesn't have a `GET` form at all (`methods=('POST',)` only) —
deleting is only ever triggered by submitting the "Delete" form in
[update.html](flaskr/templates/blog/update.html), never by just visiting a URL. This is
deliberate: a destructive action like delete shouldn't be triggerable by
simply navigating to a link (or a link preview/crawler accidentally hitting
it).

---

## 9. Templates — how the HTML is built

Flask uses the **Jinja2** templating engine. Templates live in
`flaskr/templates/`, and `render_template('blog/index.html', posts=posts)`
renders a file and injects Python data (`posts`) into it.

### `base.html` — the shared layout

```html
<!doctype html>
<title>{% block title %}{% endblock %} - Flaskr</title>
<link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
<nav>...</nav>
<section class="content">
  <header>{% block header %}{% endblock %}</header>
  {% for message in get_flashed_messages() %}
    <div class="flash">{{ message }}</div>
  {% endfor %}
  {% block content %}{% endblock %}
</section>
```

- `{% block ... %}{% endblock %}` defines a named "slot" that child templates
  can fill in. This avoids repeating the `<nav>`, `<title>`, stylesheet
  link, etc. on every single page.
- `{{ ... }}` outputs a value (Jinja2 expression syntax); `{% ... %}` runs
  logic (loops, conditionals, blocks) without outputting anything itself.
- `url_for('static', filename='style.css')` generates the correct URL for a
  static file rather than hardcoding `/static/style.css` — if the app were
  ever mounted under a different URL prefix, this still works correctly.
- `get_flashed_messages()` retrieves any messages queued up by `flash(...)`
  calls in the Python views (e.g. `flash('Username is required.')` in
  `auth.py`). Flash messages are a one-time, session-based way to show a
  message *after* a redirect (e.g. "show an error, then send the user back
  to the form").

### Child templates — e.g. `blog/index.html`

```html
{% extends 'base.html' %}

{% block header %}
  <h1>{% block title %}Posts{% endblock %}</h1>
{% endblock %}

{% block content %}
  {% for post in posts %}
    ...
  {% endfor %}
{% endblock %}
```

`{% extends 'base.html' %}` says "start from `base.html`'s structure, and
fill in these specific blocks." This is **template inheritance** — the
single most important Jinja2 concept to understand. Every page in this app
(`login.html`, `register.html`, `index.html`, `create.html`, `update.html`)
extends `base.html` and only defines the parts that are actually different
per-page.

### A subtlety worth noticing: re-populating forms after a validation error

```html
<input name="title" id="title" value="{{ request.form['title'] }}" required>
```
in [create.html](flaskr/templates/blog/create.html). If the user submits the "New Post" form
with an empty title, `create()` in `blog.py` calls `flash(error)` and
re-renders the same template — but `request.form['title']` still holds
whatever the user typed for the *body*, so the form doesn't appear to
"forget" what they entered. `update.html` does the same thing but falls
back to the original post's data if the form wasn't submitted:
`value="{{ request.form['title'] or post['title'] }}"`.

### `static/style.css`

Anything in `flaskr/static/` is served as-is at `/static/...` — no
processing, just plain files (CSS, images, JS would go here too). It's kept
separate from `templates/` because static files aren't run through Jinja2 —
they're sent to the browser byte-for-byte.

---

## 10. Config files: `pyproject.toml` vs `requirements.txt`

- **`pyproject.toml`** is the modern, standardized way to describe a Python
  project: its name, version, and dependencies (`dependencies = ["flask"]`).
  It also configures the build system (`flit_core`) used to package the
  project if you ever wanted to distribute it.
- **`requirements.txt`** is the older, simpler convention: a flat list of
  exact package versions (`Flask==3.1.3`, etc.), typically installed with
  `pip install -r requirements.txt`. It's what actually pins the exact
  versions used in this environment (Flask, Jinja2, Werkzeug, click, etc. —
  all of which are dependencies *of* Flask itself, not separately chosen).

Having both isn't unusual for a learning project — in your own future
projects, `requirements.txt` (or a tool like `pip-tools`/`uv`/`poetry`) is
usually enough; `pyproject.toml` starts to matter more once you're
packaging/distributing your code.

---

## 11. How a request actually flows through this app (worked example)

Walking through **"a logged-out visitor creates a post"** ties everything
together:

1. Visitor goes to `/create`.
2. Flask's routing matches `/create` to `blog.create`, but first runs
   `load_logged_in_user()` (registered via `before_app_request`) — since
   there's no session cookie, `g.user = None`.
3. The `@login_required` decorator wrapping `create()` checks `g.user` — it's
   `None`, so it redirects to `/auth/login` instead of ever running
   `create()`'s real code.
4. Visitor logs in: submits the login form → `POST /auth/login` → `login()`
   in `auth.py` verifies the password hash, sets
   `session['user_id']`, redirects to `/` (the `index` endpoint).
5. Visitor navigates to `/create` again. This time `load_logged_in_user()`
   finds `session['user_id']`, loads the user row into `g.user`.
   `login_required` now lets the request through.
6. `create()` renders `blog/create.html` (GET) or, on form submission
   (POST), validates the title, inserts a new row into `post` (tagged with
   `g.user['id']` as the author), commits the transaction, and redirects to
   `/` — where `index()` re-queries all posts (including the new one) and
   renders them.

---

## 12. Running the app yourself

From the project root (with dependencies installed, e.g.
`pip install -r requirements.txt`):

```
# one-time setup: create the database tables
flask --app flaskr init-db

# run the development server
flask --app flaskr run --debug
```

Then visit `http://127.0.0.1:5000` in a browser. `--debug` enables
auto-reload on code changes and gives you detailed error pages — very
useful while learning, but never use `--debug` in a real deployment (it can
leak internals to anyone hitting an error page).

---

## 13. Key Flask/Python concepts glossary

| Concept | What it means here |
|---|---|
| **Application factory** (`create_app`) | A function that builds and returns a configured `Flask` app, instead of a module-level global — enables clean testing and multiple configs. |
| **Blueprint** | A reusable, named group of routes (`auth.bp`, `blog.bp`) registered onto the app later. |
| **`g`** | Per-request scratch space for sharing data (like the DB connection or current user) between functions during one request. |
| **`current_app`** | A proxy that gives you access to the active app instance from anywhere, without passing `app` around manually. |
| **`session`** | A signed cookie-backed dict for storing small bits of data (like `user_id`) between requests for one visitor. |
| **`flash()` / `get_flashed_messages()`** | Queue a one-time message (e.g. an error) to show on the *next* page rendered. |
| **`url_for(endpoint, ...)`** | Generates a URL for a named route instead of hardcoding paths — keeps templates/redirects correct even if routes move. |
| **Decorator** (`@login_required`, `@bp.route(...)`) | A function that wraps another function to add behavior (auth checks, URL registration) without modifying its body. |
| **Parameterized SQL (`?` placeholders)** | Prevents SQL injection by never string-concatenating user input into a query. |
| **Password hashing** (`generate_password_hash` / `check_password_hash`) | Store a one-way hash of a password, never the password itself. |
| **Jinja2 template inheritance** (`{% extends %}` / `{% block %}`) | Share a common page layout across multiple templates. |
| **`teardown_appcontext`** | Register cleanup code (like closing the DB connection) that always runs after a request, even on error. |

---

## 14. Natural next steps for learning

If you want to keep building on this project as practice:

- Add a "view a single post" page (`GET /<id>`) using `get_post(id, check_author=False)`.
- Add pagination to the post list once you have more than a handful of posts.
- Write automated tests using `create_app({'TESTING': True, 'DATABASE': ...})` and Flask's test client — this is exactly what the application factory pattern was built to support.
- Move `SECRET_KEY` out of code and into an environment variable before ever deploying this anywhere public.
- Try swapping SQLite for a library like SQLAlchemy once you're comfortable with raw SQL — you'll appreciate what it's doing for you.
