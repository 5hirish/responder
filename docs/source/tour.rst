Feature Tour
============


Class-Based Views
-----------------

Class-based views (and setting some headers and stuff)::

    @api.route("/{greeting}")
    class GreetingResource:
        def on_request(self, req, resp, *, greeting):   # or on_get...
            resp.text = f"{greeting}, world!"
            resp.headers.update({'X-Life': '42'})
            resp.status_code = api.status_codes.HTTP_416


Background Tasks
----------------

Here, you can spawn off a background thread to run any function, out-of-request::

    @api.route("/")
    def hello(req, resp):

        @api.background.task
        def sleep(s=10):
            time.sleep(s)
            print("slept!")

        sleep()
        resp.content = "processing"


GraphQL
-------

Serve a GraphQL API::

    import graphene

    class Query(graphene.ObjectType):
        hello = graphene.String(name=graphene.String(default_value="stranger"))

        def resolve_hello(self, info, name):
            return f"Hello {name}"

    api.add_route("/graph", graphene.Schema(query=Query))

Visiting the endpoint will render a *GraphiQL* instance, in the browser.


Built-in Testing Client (Requests)
----------------------------------

We can then send a query to our service::

    >>> requests = api.session()
    >>> r = requests.get("http://;/graph", params={"query": "{ hello }"})
    >>> r.json()
    {'data': {'hello': 'Hello stranger'}}


Or, request YAML back::

    >>> r = requests.get("http://;/graph", params={"query": "{ hello(name:\"john\") }"}, headers={"Accept": "application/x-yaml"})
    >>> print(r.text)
    data: {hello: Hello john}

OpenAPI Schema Support
----------------------

Responder comes with built-in support for OpenAPI / marshmallow::

    import responder
    from marshmallow import Schema, fields

    api = responder.API(title="Web Service", version="1.0", openapi="3.0")


    @api.schema("Pet")
    class PetSchema(Schema):
        name = fields.Str()


    @api.route("/")
    def route(req, resp):
        """A cute furry animal endpoint.
        ---
        get:
            description: Get a random pet
            responses:
                200:
                    description: A pet to be returned
                    schema:
                        $ref = "#/components/schemas/Pet"
        """
        resp.media = PetSchema().dump({"name": "little orange"})


::

    >>> r = api.session().get("http://;/schema.yml")

    >>> print(r.text)
    components:
    parameters: {}
    schemas:
        Pet:
        properties:
            name: {type: string}
        type: object
    info: {title: Web Service, version: 1.0}
    openapi: '3.0'
    paths:
      /:
        get:
          description: Get a random pet
          responses:
            200: {description: A pet to be returned, schema: $ref = "#/components/schemas/Pet"}
    tags: []


Mount a WSGI App (e.g. Flask)
-----------------------------

Responder gives you the ability to mount another ASGI / WSGI app at a subroute::

    import responder
    from flask import Flask

    api = responder.API()
    flask = Flask(__name__)

    @flask.route('/')
    def hello():
        return 'hello'

    api.mount('/flask', flask)

That's it!

Single-Page Web Apps
--------------------

If you have a single-page webapp, you can tell Responder to serve up your ``static/index.html`` at a route, like so::

    api.add_route("/", static=True)

This will make ``index.html`` the default response to all undefined routes. Responder's CLI comes with a ``build`` command that will call ``npm run build`` for you::

    responder build

For an example of how to seamlessly integrate a React single page app with Responder check out `this project <https://github.com/metakermit/responder-react>`_.

Reading / Writing Cookies
-------------------------

Responder makes it very easy to interact with cookies from a Request, or add some to a Response::

    >>> resp.cookies["hello"] = "world"

    >>> req.cookies
    {"hello": "world"}


Using Cookie-Based Sessions
---------------------------

Responder has built-in support for cookie-based sessions. To enable cookie-based sessions, simply add something to the ``resp.session`` dictionary::

    >>> resp.session['username'] = 'kennethreitz'

A cookie called ``Responder-Session`` will be set, which contains all the data in ``resp.session``. It is signed, for verification purposes.

You can easily read a Request's session data, that can be trusted to have originated from the API::

    >>> req.session
    {'username': 'kennethreitz'}

**Note**: if you are using this in production, you should pass the ``secret_key`` argument to ``API(...)``.

WebSocket Support
-----------------

Responder supports WebSockets::

    @api.ws_route('/ws')
    async def hello(ws):
        await ws.accept()
        await ws.send_text("Hello via websocket!")
        await ws.close()


HSTS (Redirect to HTTPS)
------------------------

Want HSTS?

::

    api = responder.API(enable_hsts=True)


Boom.
