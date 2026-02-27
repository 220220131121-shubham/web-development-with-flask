# Topic 1: What Flask Actually Is (Conceptual Foundation)

Before writing any code, you must understand what problem Flask solves.

## 1. What Is a Web Application?

A web application is fundamentally:

```
Client (Browser)  →  HTTP Request  →  Server  →  HTTP Response  →  Browser
```

The browser sends a request (GET, POST, etc.).
The server processes it.
The server returns a response (HTML, JSON, file, etc.).

---

## 2. What Is Flask?

Flask is a **WSGI web framework for Python**.

Break that down:

### • WSGI

WSGI = Web Server Gateway Interface
It is a specification that defines how:

* A web server (like Gunicorn)
* Talks to
* A Python application

You do NOT need to implement WSGI yourself. Flask handles it.

---

### • Web Framework

A framework provides:

* Routing (URL → function mapping)
* Request handling
* Response generation
* Template rendering
* Extension support
* Middleware capability

Flask is called a **microframework** because:

* It provides core web functionality
* It does NOT force project structure
* It does NOT include ORM by default
* It gives flexibility instead of opinionated architecture

---

## 3. What Happens Internally When You Use Flask?

When a request comes in:

1. Web server receives HTTP request
2. WSGI forwards request to Flask app
3. Flask:

   * Matches URL against registered routes
   * Calls corresponding Python function
4. That function returns:

   * String (HTML)
   * JSON
   * Response object
5. Flask converts return value into HTTP response
6. Response is sent back to browser

This is the core invariant of all Flask apps:

> URL → Python function → Return value → HTTP response

If you understand this mapping, you understand 70% of Flask.

---

## 4. Minimal Flask Application (Anatomy)

Here is the smallest working Flask application:

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return "Hello, World!"

if __name__ == "__main__":
    app.run(debug=True)
```

Now let’s analyze it precisely.

---

### `app = Flask(__name__)`

* Creates Flask application instance.
* `__name__` tells Flask where the app is located.
* Flask uses it to locate:

  * Templates
  * Static files
  * Resources

This is not arbitrary — it matters.

---

### `@app.route("/")`

This is a decorator.

It registers the function below it as a **route handler**.

Meaning:

When a user visits:

```
http://localhost:5000/
```

Flask executes:

```python
home()
```

---

### `return "Hello, World!"`

Flask automatically converts:

* String → HTTP response
* With status code 200
* Content-Type: text/html

This implicit conversion is important.

---

### `app.run(debug=True)`

* Starts development server
* `debug=True` enables:

  * Auto reload
  * Interactive debugger
  * Better error messages

Never use debug=True in production.

---

## 5. Execution Flow (Mental Model)

Let’s simulate what happens when browser hits `/`:

1. Browser sends GET request to `/`
2. Flask sees:

   ```
   @app.route("/")
   ```
3. It calls:

   ```
   home()
   ```
4. `home()` returns string
5. Flask builds HTTP response
6. Browser displays it

---

## 6. Important Concept: Flask Is Just Python Functions

A route is just a function with metadata attached.

There is no magic.

This means:

* You can use all Python features inside routes.
* Control flow is normal Python.
* State behaves like normal Python (with concurrency considerations later).

---

## 7. What Flask Does NOT Do Automatically

Flask does NOT:

* Automatically create database models
* Automatically create admin panels
* Force MVC
* Enforce directory structure

You design architecture.

This is power and responsibility.

---

# Topic 2: Routing in Flask (Deep Dive)

Routing is the core abstraction of Flask.

Everything in Flask starts with routing.

We will examine:

1. How route matching works
2. HTTP methods
3. Dynamic URL parameters
4. URL converters
5. How Flask stores routes internally
6. Edge cases and failure modes

---

# 1️⃣ What Is a Route (Formally)?

A route is a mapping:

```
(URL rule, HTTP method) → View function
```

Important:

A route is NOT just the URL.

It is the combination of:

* Path
* HTTP method

---

## Example

```python
@app.route("/")
def home():
    return "Home"
```

This is equivalent to:

```python
@app.route("/", methods=["GET"])
```

By default, Flask allows only `GET`.

If you send a POST request to `/`, you will get:

```
405 Method Not Allowed
```

This is correct behavior.

---

# 2️⃣ HTTP Methods (Critical Concept)

The main HTTP methods you must understand:

* GET → retrieve data
* POST → submit data
* PUT → replace data
* PATCH → modify data
* DELETE → remove data

Flask differentiates routes based on method.

---

## Example: Method-Specific Routes

```python
@app.route("/data", methods=["GET"])
def get_data():
    return "GET request"

@app.route("/data", methods=["POST"])
def post_data():
    return "POST request"
```

Same URL. Different behavior.

This works because internally Flask stores:

```
("/data", GET) → get_data
("/data", POST) → post_data
```

---

# 3️⃣ Dynamic URL Parameters

Now we introduce variable parts in URLs.

Example:

```python
@app.route("/user/<username>")
def show_user(username):
    return f"User: {username}"
```

If you visit:

```
/user/shubham
```

Flask extracts:

```
username = "shubham"
```

Then calls:

```
show_user("shubham")
```

---

## Important Constraint

If route parameter name and function argument do not match:

It will raise:

```
TypeError: unexpected keyword argument
```

They must match exactly.

---

# 4️⃣ URL Converters (Type Safety at Routing Level)

By default:

```
<username>
```

is treated as string.

Flask supports converters:

```
<string:name>
<int:id>
<float:value>
<path:subpath>
<uuid:uid>
```

---

## Example: Integer Converter

```python
@app.route("/product/<int:id>")
def product(id):
    return f"Product ID: {id}"
```

If user visits:

```
/product/10  → works
/product/abc → 404
```

Important:

This does NOT call the function and then fail.

It fails during route matching.

That is efficient and correct.

---

# 5️⃣ Path Converter

```python
@app.route("/files/<path:filepath>")
def file(filepath):
    return filepath
```

`<path:>` allows slashes inside parameter.

Without it, `/files/a/b` would not match.

---

# 6️⃣ Trailing Slash Behavior

This is subtle and important.

Case 1:

```python
@app.route("/about/")
```

* Visiting `/about` → redirects to `/about/`

Case 2:

```python
@app.route("/about")
```

* Visiting `/about/` → 404

Flask enforces canonical URLs.

Be consistent in production apps.

---

# 7️⃣ How Flask Stores Routes Internally

Internally Flask uses Werkzeug’s routing system.

Each route becomes a Rule object inside:

```
app.url_map
```

You can inspect it:

```python
print(app.url_map)
```

It will output something like:

```
Map([
 <Rule '/' (GET, HEAD, OPTIONS) -> home>
])
```

This is the internal routing table.

Understanding this helps debugging complex apps later.

---

# 8️⃣ Multiple Routes for One Function

You can do:

```python
@app.route("/")
@app.route("/home")
def home():
    return "Home"
```

One function → multiple URLs.

---

# 9️⃣ Common Mistakes (Very Important)

### ❌ Forgetting methods parameter

You define POST handler but don't specify methods.

→ 405 error.

---

### ❌ Duplicate route + method combination

Flask raises assertion error at startup.

---

### ❌ Mismatch between route variable and function argument

Immediate TypeError.

---

### ❌ Using overlapping routes incorrectly

Example:

```python
@app.route("/user/<name>")
@app.route("/user/admin")
```

`/user/admin` will match the dynamic one first depending on order.

Order matters.

More specific routes should be defined first.

---

# Mental Model Summary

Routing is:

```
Incoming request
→ Flask checks URL
→ Flask checks HTTP method
→ Match Rule in url_map
→ Call function with extracted parameters
```

If no match:

* 404

If method mismatch:

* 405

---
