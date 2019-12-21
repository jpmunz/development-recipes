REST API
========

Setup
-----

* Use `Flask-RESTPLus <https://flask-restplus.readthedocs.io/en/stable/>`_
* Use `Flask-CORS <https://flask-cors.readthedocs.io/en/latest/>`_
* Use the `Application Factory Pattern <https://hackersandslackers.com/flask-application-factory/>`_

.. code:: bash

    python3 -m venv venv
    . venv/bin/activate
    pip install flask-restplus flask-CORS
    pip freeze > requirements.txt

Create ``default_config.py``:

.. code:: python

    class Config(object):
          DEBUG = False
          ERROR_404_HELP = False


    class ProductionConfig(Config):
          pass


    class DevelopmentConfig(Config):
          DEBUG = True

Create an app factory in ``myapp/__init__.py``:

.. code:: python

    import os
    from flask import Flask
    from flask_cors import CORS


    def create_app(test_config=None):
          app = Flask(__name__, instance_relative_config=True)

          if (app.config['ENV'] == 'production'):
              app.config.from_object('default_config.ProductionConfig')
          else:
              app.config.from_object('default_config.DevelopmentConfig')

          if test_config:
              app.config.from_mapping(test_config)
          else:
              app.config.from_pyfile('config.py', silent=True)

          # ensure the instance folder exists
          try:
              os.makedirs(app.instance_path)
          except OSError:
              pass

          CORS(app)

          return app

Run the API:

.. code:: bash

    . venv/bin/activate
    FLASK_APP=myapp FLASK_ENV=development flask run

.. _rest-api-db-setup:

DB Setup
--------

SQL
"""

* Use `Flask-SQLAlchemy <https://flask-sqlalchemy.palletsprojects.com/en/2.x/>`_
* Use `Flask-Migrate <https://flask-migrate.readthedocs.io/en/latest/>`_

.. code:: bash

    venv/bin/pip install Flask-SQLAlchemy Flask-Migrate
    venv/bin/pip freeze > requirements.txt

Add configuration in ``default_config.py``:

.. code:: python

    class Config(object):
        # ...
        SQLALCHEMY_DATABASE_URI = 'sqlite:////tmp/myapp.db'

Python has built-in support for SQLite and works well for small apps, for larger apps `install MySQL <https://www.digitalocean.com/community/tutorials/how-to-install-mysql-on-ubuntu-18-04>`_.

Initialize in ``app/db.py``:

.. code:: python

    from flask_sqlalchemy import SQLAlchemy

    sql = SQLAlchemy()

Hook up to the app factory in ``myapp/__init__.py``:

.. code:: python

    from flask_migrate import Migrate
    from .db import sql
    # ...

    def create_app(test_config=None):

        # ...
        sql.init_app(app)
        Migrate(app)

Use in models:

.. code:: python

    from myapp.db import sql as db

    class MyModel(db.Model):
        # ...

Generate the initial migrations folder:

.. code:: bash

    FLASK_APP=myapp venv/bin/flask db init

Generate new migrations

.. code:: bash

    FLASK_APP=myapp venv/bin/flask db migrate

Run migrations

.. code:: bash

    FLASK_APP=myapp venv/bin/flask db upgrade

Redis
"""""

* Use `flask-redis <https://pypi.org/project/flask-redis/>`_

Install Redis:

.. code:: bash

  wget http://download.redis.io/redis-stable.tar.gz
  tar xvzf redis-stable.tar.gz
  cd redis-stable
  make

Run Redis:

.. code:: bash

    redis-server

Optionally:

* `Configure and Secure Redis <https://www.digitalocean.com/community/tutorials/how-to-install-and-secure-redis-on-ubuntu-18-04#step-1-%E2%80%94-installing-and-configuring-redis>`_
* `Periodically Backup Redis Data <https://www.digitalocean.com/community/tutorials/how-to-back-up-and-restore-your-redis-data-on-ubuntu-14-04>`_

Install flask-redis:

.. code:: bash

    venv/bin/pip install flask-redis
    venv/bin/pip freeze > requirements.txt

Add configuration in ``default_config.py``:

.. code:: python

    class Config(object):
        # ...
        REDIS_URL = "redis://:@localhost:6379/0"

Initialize in ``app/db.py``:

.. code:: python

    from flask_redis import FlaskRedis

    redis_store = FlaskRedis(decode_responses=True)

Hook up to the app factory in ``myapp/__init__.py``:

.. code:: python

    from .db import redis_store
    # ...

    def create_app(test_config=None):

        # ...
        redis_store.init_app(app)

Use in models:

.. code:: python

    from myapp.db import redis_store

    # ...
    redis_store.get("foo")
    redis_store.set("foo", "bar")

Folder Structure
----------------

.. code:: bash

    myapp/
        __init__.py
        db.py
        other-shared-object-setup.py
        feature1/
            __init__.py
            endpoints.py
            models.py
            test_feature1.py
        feature2/
            ...
    ...
    instance/
    migrations/
    venv/
    default_config.py
    requirement.txt
    ...
    README.md

In general for any extension that needs to be both accessible by features and setup in the app factory create a new file
under ``myapp/`` as we did for ``db.py`` and create an instance of the extension class. The instance can then be imported in
the app factory and by features without causing circular imports.

Models
------

* Use `JSON Schema <https://json-schema.org/>`_ to outline the valid representations of the model, e.g. how it looks when it is read as json, how json requesting a write should look, etc.
* Hide DB implementation details from API endpoint logic

Example schemas:

.. code:: python

    class FooModel(object):
        read_schema = {
            '$schema': 'http://json-schema.org/draft-07/schema#',
            '$id': 'http://example.com/schemas/foo.json',
            'title': 'Foo',
            'description': 'Representation of a Foo',
            'type': 'object',
            'properties': {
                'id': {
                    'type': 'string'
                },
                'name': {
                    'type': 'string'
                }
            },
            'additionalProperties': False,
            'required': ['id', 'name'],
        }

        # 'id' is not valid in the write_schema since this is generated by the DB
        # For complex schemas reference common definitions to avoid duplication, see:
        # https://json-schema.org/understanding-json-schema/structuring.html
        write_schema = {
            '$schema': 'http://json-schema.org/draft-07/schema#',
            '$id': 'http://example.com/schemas/foo.json',
            'title': 'Foo Write',
            'description': 'Representation of a user creating or updating a Foo',
            'type': 'object',
            'properties': {
                'name': {
                    'type': 'string'
                }
            },
            'additionalProperties': False,
            'required': ['name'],
        }

        # ...

Basic pattern for a SQL-backed Model:

.. code:: python

    from myapp.db import sql as db

    class FooModel(db.Model):
        read_schema = {
            # ...
        }

        write_schema = {
            # ...
        }

        id = db.Column(db.Integer, primary_key=True)
        name = db.Column(db.String(120), nullable=False)

        @staticmethod
        def get_or_404(id):
            return FooModel.query.get_or_404(id)

        @staticmethod
        def create(from_json):
            foo = FooModel(name=from_json['name'])
            db.session.add(foo)
            db.session.commit()
            return foo

        def update(self, from_json):
            self.name = from_json['name']
            db.session.commit()
            return self

        def delete(self):
            db.session.delete(self)
            db.session.commit()

        def as_json(self):
            return {
                'id': self.id,
                'name': self.name
            }

Basic pattern for a Redis-backed Model:

.. code:: python

    import uuid
    from flask import abort
    from myapp.db import redis_store as db

    class BarModel(object):
        read_schema = {
            # ...
        }

        write_schema = {
            # ...
        }

        @staticmethod
        def key(id):
            return 'bar/{}'.format(id)

        @staticmethod
        def get_or_404(id):
            stored_value = redis_store.get(BarModel.key(id))

            if not stored_value:
                abort(404)

            return BarModel(stored_value)

        @staticmethod
        def create(from_json):
            id = uuid.uuid4().hex
            from_json['id'] = id

            stored_value = json.dumps(from_json)
            redis_store.set(BarModel.key(id), stored_value)

            return BarModel(stored_value)

        def __init__(self, stored_value):
            self.stored_value = stored_value

        def update(self, from_json):
            current = self.as_json()
            current.update(from_json)

            self.stored_value = json.dumps(current)
            redis_store.set(BarModel.key(self.as_json()['id']), self.stored_value)

            return self

        def delete(self):
            redis_store.delete(BarModel.key(self.as_json()['id']))

        def as_json(self):
            return json.loads(self.stored_value)

Endpoints
---------

* Use `Flask-RESTPLus <https://flask-restplus.readthedocs.io/en/stable/>`_
* Use `Blueprints <https://flask.palletsprojects.com/en/1.0.x/blueprints/>`_

Create an API for each distinct feature in ``<feature>/endpoints.py``:

.. code:: python

    blueprint = Blueprint('api', __name__)

    api = Api(blueprint,
              doc='/docs',
              title='Sample API',
              version='1.0',
              description='Sample API',)

Flask-RESTPlus will auto-generated a swagger-ui for the Api under the prefix specified by ``doc``.

Re-export the blueprint from ``<feature>/__init__.py``:

.. code:: python

    from .endpoints import blueprint

Then hook the blueprints up in the app factory in ``myapp/__init__.py``:

.. code:: python

    from .feature1 import blueprint as feature1_blueprint
    from .feature2 import blueprint as feature2_blueprint


    def create_app(test_config=None):

        # ...
        app.register_blueprint(feature1_blueprint, url_prefix='/feature1')
        app.register_blueprint(feature2_blueprint, url_prefix='/feature2')

This will isolate each independent set of API endpoints under their own ``url_prefix``.

Back in ``<feature>/endpoints.py`` define namespaces on url_prefixes of the api itself to help group operations related
to different resources:

.. code:: python

    foo = api.namespace('foo', description='Core operations on Foo resources')

To make full use of the auto-generated docs create a schema model for each representation of the model:

.. code:: python

    foo_read_schema_model = foo.schema_model('Foo', FooModel.read_schema)
    foo_write_schema_model = foo.schema_model('Write Foo', FooModel.write_schema)

Then supply them to the ``expect`` and ``response`` decorators on resource methods. Basic pattern for a set of CRUD
resources:

.. code:: python

    class FooResource(Resource):
        @staticmethod
        def validate_write_request(value):
            try:
                jsonschema.validate(request.json, FooModel.write_schema)
            except jsonschema.ValidationError as e:
                abort(400, e.message)


    @foo.route("/")
    class FooList(FooResource):
        @foo.expect(foo_write_schema_model)
        @foo.response(201, 'Created', foo_read_schema_model)
        @foo.response(400, description='Invalid Foo')
        def post(self):
            """
            Creates a new Foo
            """

            self.validate_write_request(request.json)
            foo = FooModel.create(request.json)

            return foo.as_json(), 201


    @foo.route("/<string:id>")
    class Foo(FooResource):
        @foo.response(200, 'Success', foo_read_schema_model)
        @foo.response(404, description='Not Found')
        def get(self, id):
            """
            Gets a Foo
            """

            foo = FooModel.get_or_404(id)

            return foo.as_json()

        @foo.expect(foo_write_schema_model)
        @foo.response(200, 'Success', foo_read_schema_model)
        @foo.response(400, description='Invalid Foo')
        @foo.response(404, description='Not Found')
        def put(self, id):
            """
            Updates a Foo
            """

            foo = FooModel.get_or_404(id)

            self.validate_write_request(request.json)
            foo.update(request.json)

            return foo.as_json()

        @foo.response(204, description='No Content')
        @foo.response(404, description='Not Found')
        def delete(self, id):
            """
            Deletes a Foo
            """

            foo = FooModel.get_or_404(id)

            foo.delete()

            return '', 204

Unit Tests
----------

    * Use `pytest <https://docs.pytest.org/en/latest/>`_
    * Don't mock the DB, use a real isolated instance, see `this <https://8thlight.com/blog/eric-smith/2011/10/27/thats-not-yours.html>`_ for motivation

.. code:: bash

    venv/bin/pip install pytest
    venv/bin/pip freeze > requirements.txt

Create ``conftest.py`` for global fixture setup:

Using SQL:

.. code:: python

    import pytest
    from myapp import create_app, sql


    def create_test_app():
        return create_app(test_config=dict(
            SQLALCHEMY_DATABASE_URI='sqlite:////tmp/myapp_test.db',
            TESTING=True,
        ))


    # Automatically applied to all tests, scoped to 'session' so full setup and teardown
    # of the DB only happens once per test run
    @pytest.fixture(autouse=True, scope='session')
    def sql_cleanup():
        # Commands require an app context, create a temporary one
        sql.create_all(app=create_test_app())
        yield
        sql.drop_all(app=create_test_app())


    @pytest.fixture(scope='function')
    def client():
        app = create_test_app()
        with app.test_client() as client:
            # Wrap each test run in a transaction that is rolled back on teardown
            # This allows each test to run against a fresh DB without the overhead
            # of dropping and re-creating all the tables
            # See http://alexmic.net/flask-sqlalchemy-pytest
            with app.app_context():
                connection = sql.engine.connect()
                transaction = connection.begin()
                options = dict(bind=connection, binds={})
                session = sql.create_scoped_session(options=options)

            sql.session = session

            yield client

            transaction.rollback()
            connection.close()
            session.remove()

Using Redis:

.. code:: python

    import pytest
    from myapp import create_app, redis_store

    def create_test_app():
        return create_app(test_config=dict(
            # Specify a different DB number to isolate from development DB
            REDIS_URL="redis://:@localhost:6379/1",
            TESTING=True,
        ))


    @pytest.fixture(autouse=True, scope='session')
    def redis_cleanup():
        yield
        redis_store.flushdb()


    @pytest.fixture(scope='function')
    def client():
        app = create_test_app()
        with app.test_client() as client:
            yield client

Add tests to ``feature1/test_feature1.py``:

.. code:: python

    def test_create_bar(client):
        res = client.post('/v1/bar/', json={
            'name': 'my bar'
        })
        assert res.status_code == 201

        created_bar = res.get_json()
        expected_bar = {'id': created_bar['id'], 'name': 'my bar'}
        assert created_bar == expected_bar

        # Assert that the resource was actually created
        res = client.get('/v1/bar/{}'.format(created_bar['id']))
        assert res.status_code == 200
        assert res.get_json() == expected_bar

Test each possible response path for each endpoint. If the JSON representation of the resource is complex test multiple
cases with `@pytest.mark.parametrize <https://docs.pytest.org/en/latest/parametrize.html>`_.

Models don't need to be tested in isolation since the endpoints will exercise them, though if they have
complex logic it may be helpful for debugging purposes.

Run tests:

.. code:: bash

    venv/bin/pytest

Code Style
----------

* Use `Black <https://black.readthedocs.io/en/stable/>`_
* Use `Flake8 <http://flake8.pycqa.org/en/latest/>`_
* Use `Pre-commit <https://pre-commit.com/>`_
* See `this <https://ljvmiranda921.github.io/notebook/2018/06/21/precommits-using-black-and-flake8/>`_ for motvation

.. code:: bash

    venv/bin/pip install pre-commit black flake8
    venv/bin/pip freeze > requirements.txt

Configure ``.flake8`` to work with black:

.. code:: ini

    [flake8]
    max-line-length = 88

Configure the pre-commit hook in ``.pre-commit-config.yaml``:

.. code:: yaml

    repos:
    -   repo: https://github.com/ambv/black
        rev: stable
        hooks:
        - id: black
          language_version: python3.6
    -   repo: https://github.com/pre-commit/pre-commit-hooks
        rev: v1.2.3
        hooks:
        - id: flake8
          exclude: migrations

Install the pre-commit hook:

.. code:: bash

    venv/bin/pre-commit install

Adding Dependencies
-------------------

.. code:: bash

    venv/bin/pip install <dependency>
    venv/bin/pip freeze > requirements.txt

Install dependencies:

.. code:: bash

    venv/bin/pip install -r requirements.txt

Deployment
----------

* Setup hosting for a :ref:`wsgi-app-hosting`

Copy over the code:

.. code:: bash

    rsync -avzr --delete --exclude '__pycache__*' myapp.ini default_config.py wsgi.py myapp requirements <server>/var/app/myapp

Build the dependencies:

.. code:: bash

    ssh -t <server> 'cd /var/app/myapp && python3 -m venv venv && venv/bin/pip install --upgrade pip && venv/bin/pip install -r requirements.txt'

Make sure that any required config overrides are set in ``<server>/var/app/myapp/instance/config.py``:

.. code:: python

    SECRET_PASS = # ...

Run any migrations if using SQL:

.. code:: bash

    FLASK_APP=myapp venv/bin/flask db upgrade

Restart the app service:

.. code:: bash

    ssh -t <server> 'systemctl restart myapp'

CI/CD
-----

* Use `GitHub Actions <https://help.github.com/en/actions/automating-your-workflow-with-github-actions>`_

Generate a password-less SSH key and copy over the public key to ``.ssh/authorized_keys`` on the server being deployed to.
In the GitHub repo for the project add a DEPLOY_KEY secret and paste in the private key then add the following secrets:

* DEPLOY_DESTINATION: ``<username>@<server>:/var/app/myapp``
* DEPLOY_USERNAME: ``<username>``
* DEPLOY_HOST: ``<server>``

By default remotely restarting the app service requires a password entry, in order to do this automatically through CD
allow it to be executed without a password:

.. code:: bash

    sudo visudo

And add this line to the sudoers file:

.. code:: bash

    # ...
    <username> ALL = NOPASSWD: /bin/systemctl restart myapp.service

Create a ``.github/workflows/deploy.yml`` action in the repo:

.. code:: yaml

    name: Build and Deploy

    on:
      push:
        branches:
          - master

    jobs:
      build:
        runs-on: ubuntu-latest
        steps:
        - uses: actions/checkout@v1
        - name: Setup Redis for use in testing
          uses: shogo82148/actions-setup-redis@v1
          with:
            redis-version: '5.x'
        - name: Redis ping
          run: redis-cli ping
        - name: Set up Python 3.7
          uses: actions/setup-python@v1
          with:
            python-version: 3.7
        - name: Install dependencies
          run: |
            python -m venv venv
            venv/bin/pip install --upgrade pip
            venv/bin/pip install -r requirements.txt
        - name: Lint and test
          run: |
            venv/bin/flake8 . --exclude venv
            venv/bin/pytest
        - name: Deploy the build
          id: deploy
          uses: Pendect/action-rsyncer@v1.1.0
          env:
            DEPLOY_KEY: ${{secrets.DEPLOY_KEY}}
          with:
            flags: '-avzr --delete'
            options: ''
            ssh_options: ''
            src: 'myapp.ini default_config.py wsgi.py myapp venv'
            dest: ${{ secrets.DEPLOY_DESTINATION }}
        - name: Display status from deploy
          run: echo "${{ steps.deploy.outputs.status }}"
        - name: Restart the app
          uses: appleboy/ssh-action@master
          with:
            host: ${{ secrets.DEPLOY_HOST }}
            username: ${{ secrets.DEPLOY_USERNAME }}
            key: ${{ secrets.DEPLOY_KEY }}
            script: |
              cd /var/app/myapp
              python3 -m venv venv
              venv/bin/pip install --upgrade pip
              venv/bin/pip install -r requirements.txt
              sudo /bin/systemctl restart myapp.service

Push to the repo to trigger the action

TL;DR
-----

See `this repo <https://github.com/jpmunz/sample-rest-api>`_ for an example project that encapsulates these tips.

References
----------

* `Flask <https://flask.palletsprojects.com/en/1.1.x/>`_
* `Application Factory Pattern <https://hackersandslackers.com/flask-application-factory/>`_
* `Flask-RESTPLus <https://flask-restplus.readthedocs.io/en/stable/>`_
* `Flask-SQLAlchemy <https://flask-sqlalchemy.palletsprojects.com/en/2.x/>`_
* `Flask-Migrate <https://flask-migrate.readthedocs.io/en/latest/>`_
* `flask-redis <https://pypi.org/project/flask-redis/>`_
* `Redis <https://redis.io/documentation>`_
* `pytest <https://docs.pytest.org/en/latest/>`_
* `That's Not Yours <https://8thlight.com/blog/eric-smith/2011/10/27/thats-not-yours.html>`_
* `Delightful testing with pytest and Flask-SQLAlchemy <http://alexmic.net/flask-sqlalchemy-pytest>`_
* `Automate Python workflow using pre-commits <https://ljvmiranda921.github.io/notebook/2018/06/21/precommits-using-black-and-flake8/>`_
