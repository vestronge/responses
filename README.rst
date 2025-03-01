Responses
=========

.. image:: https://img.shields.io/pypi/v/responses.svg
    :target: https://pypi.python.org/pypi/responses/

..  image:: https://travis-ci.org/getsentry/responses.svg?branch=master
    :target: https://travis-ci.org/getsentry/responses

.. image:: https://img.shields.io/pypi/pyversions/responses.svg
    :target: https://pypi.org/project/responses/

A utility library for mocking out the ``requests`` Python library.

..  note::

    Responses requires Python 2.7 or newer, and requests >= 2.0


Installing
----------

``pip install responses``


Basics
------

The core of ``responses`` comes from registering mock responses:

..  code-block:: python

    import responses
    import requests

    @responses.activate
    def test_simple():
        responses.add(responses.GET, 'http://twitter.com/api/1/foobar',
                      json={'error': 'not found'}, status=404)

        resp = requests.get('http://twitter.com/api/1/foobar')

        assert resp.json() == {"error": "not found"}

        assert len(responses.calls) == 1
        assert responses.calls[0].request.url == 'http://twitter.com/api/1/foobar'
        assert responses.calls[0].response.text == '{"error": "not found"}'

If you attempt to fetch a url which doesn't hit a match, ``responses`` will raise
a ``ConnectionError``:

..  code-block:: python

    import responses
    import requests

    from requests.exceptions import ConnectionError

    @responses.activate
    def test_simple():
        with pytest.raises(ConnectionError):
            requests.get('http://twitter.com/api/1/foobar')

Lastly, you can pass an ``Exception`` as the body to trigger an error on the request:

..  code-block:: python

    import responses
    import requests

    @responses.activate
    def test_simple():
        responses.add(responses.GET, 'http://twitter.com/api/1/foobar',
                      body=Exception('...'))
        with pytest.raises(Exception):
            requests.get('http://twitter.com/api/1/foobar')


Response Parameters
-------------------

Responses are automatically registered via params on ``add``, but can also be
passed directly:

..  code-block:: python

    import responses

    responses.add(
        responses.Response(
            method='GET',
            url='http://example.com',
        )
    )

The following attributes can be passed to a Response mock:

method (``str``)
    The HTTP method (GET, POST, etc).

url (``str`` or compiled regular expression)
    The full resource URL.

match_querystring (``bool``)
    Include the query string when matching requests.
    Enabled by default if the response URL contains a query string,
    disabled if it doesn't or the URL is a regular expression.

body (``str`` or ``BufferedReader``)
    The response body.

json
    A Python object representing the JSON response body. Automatically configures
    the appropriate Content-Type.

status (``int``)
    The HTTP status code.

content_type (``content_type``)
    Defaults to ``text/plain``.

headers (``dict``)
    Response headers.

stream (``bool``)
    Disabled by default. Indicates the response should use the streaming API.

match (``list``)
    A list of callbacks to match requests based on request body contents.


Matching Request Parameters
---------------------------

When adding responses for endpoints that are sent request data you can add
matchers to ensure your code is sending the right parameters and provide
different responses based on the request body contents. Responses provides
matchers for JSON and URLencoded request bodies and you can supply your own for
other formats.

.. code-block:: python

    import responses
    import requests

    @responses.activate
    def test_calc_api():
        responses.add(
            responses.POST,
            url='http://calc.com/sum',
            body="4",
            match=[
                responses.urlencoded_params_matcher({"left": "1", "right": "3"})
            ]
        )
        requests.post("http://calc.com/sum", data={"left": 1, "right": 3})

Matching JSON encoded data can be done with ``responses.json_params_matcher()``.
If your application uses other encodings you can build your own matcher that
returns ``True`` or ``False`` if the request parameters match. Your matcher can
expect a ``request_body`` parameter to be provided by responses.

Dynamic Responses
-----------------

You can utilize callbacks to provide dynamic responses. The callback must return
a tuple of (``status``, ``headers``, ``body``).

..  code-block:: python

    import json

    import responses
    import requests

    @responses.activate
    def test_calc_api():

        def request_callback(request):
            payload = json.loads(request.body)
            resp_body = {'value': sum(payload['numbers'])}
            headers = {'request-id': '728d329e-0e86-11e4-a748-0c84dc037c13'}
            return (200, headers, json.dumps(resp_body))

        responses.add_callback(
            responses.POST, 'http://calc.com/sum',
            callback=request_callback,
            content_type='application/json',
        )

        resp = requests.post(
            'http://calc.com/sum',
            json.dumps({'numbers': [1, 2, 3]}),
            headers={'content-type': 'application/json'},
        )

        assert resp.json() == {'value': 6}

        assert len(responses.calls) == 1
        assert responses.calls[0].request.url == 'http://calc.com/sum'
        assert responses.calls[0].response.text == '{"value": 6}'
        assert (
            responses.calls[0].response.headers['request-id'] ==
            '728d329e-0e86-11e4-a748-0c84dc037c13'
        )

You can also pass a compiled regex to ``add_callback`` to match multiple urls:

..  code-block:: python

    import re, json

    from functools import reduce

    import responses
    import requests

    operators = {
      'sum': lambda x, y: x+y,
      'prod': lambda x, y: x*y,
      'pow': lambda x, y: x**y
    }

    @responses.activate
    def test_regex_url():

        def request_callback(request):
            payload = json.loads(request.body)
            operator_name = request.path_url[1:]

            operator = operators[operator_name]

            resp_body = {'value': reduce(operator, payload['numbers'])}
            headers = {'request-id': '728d329e-0e86-11e4-a748-0c84dc037c13'}
            return (200, headers, json.dumps(resp_body))

        responses.add_callback(
            responses.POST,
            re.compile('http://calc.com/(sum|prod|pow|unsupported)'),
            callback=request_callback,
            content_type='application/json',
        )

        resp = requests.post(
            'http://calc.com/prod',
            json.dumps({'numbers': [2, 3, 4]}),
            headers={'content-type': 'application/json'},
        )
        assert resp.json() == {'value': 24}

    test_regex_url()


If you want to pass extra keyword arguments to the callback function, for example when reusing
a callback function to give a slightly different result, you can use ``functools.partial``:

.. code-block:: python

    from functools import partial

    ...

        def request_callback(request, id=None):
            payload = json.loads(request.body)
            resp_body = {'value': sum(payload['numbers'])}
            headers = {'request-id': id}
            return (200, headers, json.dumps(resp_body))

        responses.add_callback(
            responses.POST, 'http://calc.com/sum',
            callback=partial(request_callback, id='728d329e-0e86-11e4-a748-0c84dc037c13'),
            content_type='application/json',
        )


You can see params passed in the original ``request`` in ``responses.calls[].request.params``:

.. code-block:: python

    import responses
    import requests

    @responses.activate
    def test_request_params():
        responses.add(
            method=responses.GET,
            url="http://example.com?hello=world",
            body="test",
            match_querystring=False,
        )

        resp = requests.get('http://example.com', params={"hello": "world"})
        assert responses.calls[0].request.params == {"hello": "world"}

Responses as a context manager
------------------------------

..  code-block:: python

    import responses
    import requests

    def test_my_api():
        with responses.RequestsMock() as rsps:
            rsps.add(responses.GET, 'http://twitter.com/api/1/foobar',
                     body='{}', status=200,
                     content_type='application/json')
            resp = requests.get('http://twitter.com/api/1/foobar')

            assert resp.status_code == 200

        # outside the context manager requests will hit the remote server
        resp = requests.get('http://twitter.com/api/1/foobar')
        resp.status_code == 404

Responses as a pytest fixture
-----------------------------

.. code-block:: python

    @pytest.fixture
    def mocked_responses():
        with responses.RequestsMock() as rsps:
            yield rsps

    def test_api(mocked_responses):
        mocked_responses.add(
            responses.GET, 'http://twitter.com/api/1/foobar',
            body='{}', status=200,
            content_type='application/json')
        resp = requests.get('http://twitter.com/api/1/foobar')
        assert resp.status_code == 200

Responses inside a unittest setUp()
-----------------------------------

When run with unittest tests, this can be used to set up some
generic class-level responses, that may be complemented by each test

.. code-block:: python

    def setUp():
        self.responses = responses.RequestsMock()
        self.responses.start()

        # self.responses.add(...)

        self.addCleanup(self.responses.stop)
        self.addCleanup(self.responses.reset)

    def test_api(self):
        self.responses.add(
            responses.GET, 'http://twitter.com/api/1/foobar',
            body='{}', status=200,
            content_type='application/json')
        resp = requests.get('http://twitter.com/api/1/foobar')
        assert resp.status_code == 200

Assertions on declared responses
--------------------------------

When used as a context manager, Responses will, by default, raise an assertion
error if a url was registered but not accessed. This can be disabled by passing
the ``assert_all_requests_are_fired`` value:

.. code-block:: python

    import responses
    import requests

    def test_my_api():
        with responses.RequestsMock(assert_all_requests_are_fired=False) as rsps:
            rsps.add(responses.GET, 'http://twitter.com/api/1/foobar',
                     body='{}', status=200,
                     content_type='application/json')

assert_call_count
-----------------

Assert that the request was called exactly n times.

.. code-block:: python

    import responses
    import requests

    @responses.activate
    def test_assert_call_count():
        responses.add(responses.GET, "http://example.com")

        requests.get("http://example.com")
        assert responses.assert_call_count("http://example.com", 1) is True

        requests.get("http://example.com")
        with pytest.raises(AssertionError) as excinfo:
            responses.assert_call_count("http://example.com", 1)
        assert "Expected URL 'http://example.com' to be called 1 times. Called 2 times." in str(excinfo.value)


Multiple Responses
------------------

You can also add multiple responses for the same url:

..  code-block:: python

    import responses
    import requests

    @responses.activate
    def test_my_api():
        responses.add(responses.GET, 'http://twitter.com/api/1/foobar', status=500)
        responses.add(responses.GET, 'http://twitter.com/api/1/foobar',
                      body='{}', status=200,
                      content_type='application/json')

        resp = requests.get('http://twitter.com/api/1/foobar')
        assert resp.status_code == 500
        resp = requests.get('http://twitter.com/api/1/foobar')
        assert resp.status_code == 200


Using a callback to modify the response
---------------------------------------

If you use customized processing in `requests` via subclassing/mixins, or if you
have library tools that interact with `requests` at a low level, you may need
to add extended processing to the mocked Response object to fully simulate the
environment for your tests.  A `response_callback` can be used, which will be
wrapped by the library before being returned to the caller.  The callback
accepts a `response` as it's single argument, and is expected to return a
single `response` object.

..  code-block:: python

    import responses
    import requests

    def response_callback(resp):
        resp.callback_processed = True
        return resp

    with responses.RequestsMock(response_callback=response_callback) as m:
        m.add(responses.GET, 'http://example.com', body=b'test')
        resp = requests.get('http://example.com')
        assert resp.text == "test"
        assert hasattr(resp, 'callback_processed')
        assert resp.callback_processed is True


Passing through real requests
-----------------------------

In some cases you may wish to allow for certain requests to pass through responses
and hit a real server. This can be done with the ``add_passthru`` methods:

.. code-block:: python

    import responses

    @responses.activate
    def test_my_api():
        responses.add_passthru('https://percy.io')

This will allow any requests matching that prefix, that is otherwise not registered
as a mock response, to passthru using the standard behavior.

Regex can be used like:

.. code-block:: python

    responses.add_passthru(re.compile('https://percy.io/\\w+'))


Viewing/Modifying registered responses
--------------------------------------

Registered responses are available as a public method of the RequestMock
instance. It is sometimes useful for debugging purposes to view the stack of
registered responses which can be accessed via ``responses.registered()``.

The ``replace`` function allows a previously registered ``response`` to be
changed. The method signature is identical to ``add``. ``response`` s are
identified using ``method`` and ``url``. Only the first matched ``response`` is
replaced.

..  code-block:: python

    import responses
    import requests

    @responses.activate
    def test_replace():

        responses.add(responses.GET, 'http://example.org', json={'data': 1})
        responses.replace(responses.GET, 'http://example.org', json={'data': 2})

        resp = requests.get('http://example.org')

        assert resp.json() == {'data': 2}


The ``upsert`` function allows a previously registered ``response`` to be
changed like ``replace``. If the response is registered, the ``upsert`` function
will registered it like ``add``.

``remove`` takes a ``method`` and ``url`` argument and will remove **all**
matched responses from the registered list.

Finally, ``reset`` will reset all registered responses.

Contributing
------------

Responses uses several linting and autoformatting utilities, so it's important that when
submitting patches you use the appropriate toolchain:

Clone the repository:

.. code-block:: shell

    git clone https://github.com/getsentry/responses.git

Create an environment (e.g. with ``virtualenv``):

.. code-block:: shell

    virtualenv .env && source .env/bin/activate

Configure development requirements:

.. code-block:: shell

    make develop

Responses uses `Pytest <https://docs.pytest.org/en/latest/>`_ for
testing. You can run all tests by:

.. code-block:: shell

    pytest

And run a single test by:

.. code-block:: shell

    pytest -k '<test_function_name>'
