Web App
=======

Design
------

* Start with paper
* See `Describing UI state with data <http://nicolashery.com/describing-ui-state-with-data/>`_ for a methodical approach to UI design

1. Drawn out the UI with pen and paper
2. Make a list of 'What the user can see', this is the ``state`` of the app
3. Make a list of 'What the user can do', these are the ``state transitions`` of the app
4. Separate anything in the ``state`` that can be derived from something else, these are the ``projections``

Setup
-----

* Use `Create React App <https://create-react-app.dev/docs/getting-started/>`_

.. code:: bash

    npx create-react-app <project>
    cd <project>
    npm start

State Model
-----------

* Use `Redux Toolkit <https://redux-toolkit.js.org/introduction/quick-start>`_

.. code:: bash

    npm install --save-exact @reduxjs/toolkit redux-logger

Create ``state/index.js`` then for each:

* piece of core ``state``: create a slice and an initial state
* ``state transition``: create a reducer function
* ``projection``: create a selector

.. code:: javascript

    import { createAction, createSlice } from "@reduxjs/toolkit";
    import { combineReducers } from "redux";

    /*
      REDUCERS
    */

    const fooSlice = createSlice({
      name: "foo",
      initialState: {  },
      reducers: {
        action1(state, action) {
          state.bar = action.payload.bar;
          // ...
        },
        // ...
    });

    export const { action1 } = fooSlice.actions;
    // ...

    export const rootReducer = combineReducers({
      foo: fooSlice.reducer,
      // ...
    });

    /*
      SELECTORS
    */

    export function selector1(state) {
      return state.foo;
    }

    // ...

In ``index.js`` import the ``rootReducer`` and setup the App:

.. code:: jsx

    import React from "react"
    import ReactDOM from "react-dom"
    import { configureStore, getDefaultMiddleware } from "@reduxjs/toolkit"
    import logger from "redux-logger"
    import { Provider} from "react-redux"
    import { rootReducer } from "src/state"
    import App from "src/views/App"

    const store = configureStore({
      reducer: rootReducer,
      middleware: [...getDefaultMiddleware(), logger],
    });

    ReactDOM.render(
      <Provider store={store}>
        <App />
      </Provider>,
      document.getElementById('root')
    );


``redux-toolkit`` handles state transitions behind the scenes by using `Immer`_. If you need to do mutations in order to
derive something from the state you can use it directly:

.. code:: bash

    npm install --save-exact @reduxjs/toolkit redux-logger

.. code:: javascript

    import { produce } from "immer";

    const derivedState = produce(state, draft => {
        // Mutate draft
    });


Controlled Components
---------------------

* Use `React Redux <https://react-redux.js.org/introduction/quick-start>`_
* Use ``connect`` to map application state and state transition methods to props
* Use ``autoBind`` in the constructor to make sure DOM handlers have access to ``this``
* Export both the connected and unconnected component to facilitate unit testing

.. code:: bash

    npm install --save-exact react-redux react-autobind

Example:

.. code:: jsx

    import React from "react";
    import { connect } from "react-redux";
    import { selector1, selector2, action1, action2 } from "src/state";
    import autoBind from "react-autobind";

    export class Component extends React.Component {
      constructor(props) {
        super(props);
        this.state = {};
        autoBind(this);
      }

      handleChange(event) {
        this.setState({foo: event.target.value});
      }

      handleClick() {
        this.props.action1({value: this.state.foo});
      }

      render() {
        return (
          <div className="Component">
            <div>{this.prop1}</div>
            <input type="text" onChange={this.handleChange} />
            <button onClick={this.handleClick}>Click</button>
          </div>
        );
      }
    }

    export default connect((state, props) => {
      return {
        prop1: selector1(state),
        prop2: selector2(state),
        // ...
      }
    }, { action1, action2 })(Component);

Routes
------

* Use `React DOM Router <https://reacttraining.com/react-router/web/guides/quick-start>`_

Setup routes in ``App.js`` using ``react-router-dom``:

.. code:: bash

    npm install --save-exact react-router-dom

.. code:: javascript

    import React from "react";
    import { Switch, Route } from "react-router-dom";
    import Home from "./Home";
    import Foo from "./Foo";

    class App extends React.Component {
      render() {
        return (
          <div className="App">
            <Switch>
              <Route exact path="/">
                <Home />
              </Route>
              <Route path="/foo/:id" render={(props) => {
                return <Foo id={props.match.params.id} />
              }} />
            </Switch>
          </div>
        );
      }
    }

    export default App;

In ``index.js`` wrap the App component in a Router:

.. code:: javascript

    import { BrowserRouter } from "react-router-dom";
    // ...

    ReactDOM.render(
      <Provider store={store}>
        <BrowserRouter>
          <App />
        </BrowserRouter>
      </Provider>,
      document.getElementById("root")
    );

Async Operations
----------------

* Use `Redux Saga <https://redux-saga.js.org/>`_
* Separate async operations into layers:

    1. Views make requests to trigger async operations and render progress, error, or success UIs based on the app state
    2. Middleware watches for requests, fires off api calls, and makes state transitions based on the results
    3. API code performs the actual calls to external services

Create a separate ``api`` module to perform the async calls:

.. code:: javascript

    export default class API {
      _fetch(url, options) {
        return fetch(process.env.REACT_APP_API_HOST + url, Object.assign({
          // Set options common to all your API calls here
          headers: {
            "Content-Type": "application/json"
          }
        }, options)).then(response => {
          if (!response.ok) {
            throw Error(response.statusText);
          }

          return response.json();
        });
      }

      apiMethod(args) {
        return this._fetch("/v1/endpoint");
      }
    };

And have an instance available in ``src/state/index.js``:

.. code:: javascript

    import API from "src/api";

    const api = new API();

Use variables in ``.env.development``, ``env.production``, ``.env.test`` to control the location of the API in different
environments:

.. code:: bash

    REACT_APP_API_HOST=http://localhost:5000

Create a slice in your app state dedicated to tracking the flow of async operations along with a `getAsync` selector:

.. code:: javascript

    const asycnRequestsSlice = createSlice({
      name: "async",
      initialState: {},
      reducers: {
        start(state, action) {
          state[action.payload.id] = { inProgress: true };
        },
        error(state, action) {
          state[action.payload.id] = {
            inProgress: false,
            error: action.payload.error
          };
        },
        success(state, action) {
          state[action.payload.id] = {
            success: true,
            response: action.payload.response
          };
        }
      }
    });

    export const rootReducer = combineReducers({
      async: asycnRequestsSlice.reducer,
      // ...
    });

    export function getAsync(state, id) {
      return state.async[id] || {};
    }

Create a standalone action to triggering each async operation. The action payload should include at least an ``id``
so that the operation can be tracked:

.. code:: javascript

    export const someAsyncRequest = createAction("someAsyncRequest");

Install `Redux Saga <https://redux-saga.js.org/>`_:

.. code:: bash

    npm install --save-exact redux-saga

Create a rootSaga for handling the async requests:

.. code:: javascript

    import { put, call, takeLatest } from "redux-saga/effects";
    // ...

    export const rootSaga = function*() {
      yield takeLatest(someAsyncRequest.type, handleSomeAsyncRequest);
      // ...
    };

The handlers should follow the same basic pattern:

1. Do a state transition to indicate that the operation has begun
2. Make an api call to actually perform the operation
3. Wait for the api call to complete and do either a success or error state transition for the operation

.. code:: javascript

    const handleSomeAsyncRequest = function*(action) {
      const {id} = action.payload
      try {
        yield put(asyncRequestsSlice.actions.start({ id }));
        const response = yield call([api, "apiMethod"], apiArg1, apiArg2);
        yield put(someStateTransitionBasedOnTheResponse({ response }));
        yield put(asyncRequestsSlice.actions.success({ id, response }));
      } catch (error) {
        yield put(asyncRequestsSlice.actions.error({ id, error: error.toString() }));
      }
    };

Finally, hook up the root saga in ``index.js``:

.. code:: javascript

    import createSagaMiddleware from "redux-saga"
    import { configureStore, getDefaultMiddleware } from "@reduxjs/toolkit"
    import logger from "redux-logger"
    import { rootReducer, rootSaga } from "src/state"

    const sagaMiddleware = createSagaMiddleware();

    const store = configureStore({
      reducer: rootReducer,
      middleware: [...getDefaultMiddleware(), sagaMiddleware, logger],
    });

    sagaMiddleware.run(rootSaga);

Controlled Components can then use async actions and the ``getAsync`` selector to alter rendering based on the state of
the async operations:

.. code:: jsx

    import React from "react";
    import { connect } from "react-redux";
    import { someAsyncRequest, barAsyncRequest, getAsync } from "src/state"

    class Edit extends React.Component {
      componentDidMount() {
        this.props.someAsyncRequest({id: this.props.requestId});
      }

      render() {
        if (this.props.request.inProgress) {
          return <div>Loading</div>;
        } else if (this.props.request.error) {
          return <div>{"Some error: " + this.props.request.error}</div>;
        } else {
          return <div>Request Complete</div>;
        }
      }
    }

    export default connect((state, props) => {
      return {
        request: getAsync(state, props.requestId),
      }
    }, {someAsyncRequest})(Edit);


Style
-----

* Use scss
* Use `Bootstrap <https://getbootstrap.com/>`_
* Use `React Bootstrap <https://react-bootstrap.github.io/getting-started/introduction>`_
* Co-locate ``.scss`` files with Views so that the style can be imported with ``import ./View.scss``
* The top-level tag in a Component should have ``className="Component"`` and the scss should begin with ``.Component {`` to limit scope
* Include a ``mixins.scss`` with common helpers

.. code:: bash

    npm install --save-exact bootstrap node-sass react-bootstrap

From ``index.js`` do ``import './index.scss'`` then in ``index.scss``:

.. code:: scss

    @import "variables";
    @import "~bootstrap/scss/bootstrap";

    /* Global Styles */
    body {

    }

``variables.scss`` should include any `bootstrap variable <https://getbootstrap.com/docs/4.0/getting-started/theming/#introduction>`_ overrides:

.. code:: scss

    $primary: #FF0000;
    $accent: #00FF00;
    $black: #222;

Folder Structure
----------------

* Co-locate tests with source files
* Co-locate styles with views

.. code::

    public/
    src/
        api/
            index.js
            api.test.js
        assets/
            foo.svg
             ...
        state/
            index.js
            state.test.js
        views/
            App.js
            App.scss
            App.test.js
            View2.js
            View2.scss
            View2.test.js
            ...
        index.js
        index.scss
        setupTests.js
        someConstants.js
        ...
    package.json
    README.md
    ...

Provides a clean separation between the main layers of the app. Files in ``views`` will depend on files in ``state`` which will
depend on files in ``api`` but ideally never in the reverse direction. Feature folders can be added to organize larger projects,
either at the top-level:

.. code::

    src/
        feature1/
            api/
            assets/
            state/
            views/
        feature2/
            api/
            assets/
            state/
            views/
        ...

Or within a given layer:

.. code:: bash

    api/
    assets/
    state/
    views/
        feature1/
        feature2/
        ...

If the top-level ``api/index.js``/``state/index.js`` files get too large they can be split into logical chunks, e.g.
``state/additional.js``:

.. code:: javascript

    export function someAdditionalSelector(state) { /* ... */ }

Then re-export the function in ``state/index.js`` so that consumers don't need to be aware of the re-organization:

.. code:: javascript

    import { someAdditionalSelector } from "./additional";

    // ...

    export { someAdditionalSelector }

For imports across directories allow relative imports in ``jsconfig.json``:

.. code:: json

    {
      "compilerOptions": {
        "baseUrl": "."
      }
    }

Then import using ``from 'src/<module>'``. Always include ``src`` unless it's a relative import from the same directory,
e.g. ``from './<module>`` to differentiate app modules from external packages.

Unit Tests
----------

* Use `Jest <https://jestjs.io/docs/en/getting-started>`_
* Use `React Testing Library <https://testing-library.com/docs/react-testing-library/intro>`_

Both React and Redux encourage building components that leverage pure, stateless functions. The same methodology should
extend to unit testing, ideally in a way that approaches `Table Driven Tests <https://github.com/golang/go/wiki/TableDrivenTests>`_ as much as possible.

Testing Controlled Components
""""""""""""""""""""""""""""""
    * Given: a set of properties and a set of user interactions
    * Expect: a particular rendered HTML and a set of resulting calls on property methods
    * Use `Jest Snapshot Testing <https://jestjs.io/docs/en/snapshot-testing>`_ for assertions against rendered HTML
    * Use `React Testing Library <https://testing-library.com/docs/react-testing-library/intro>`_ for firing events
    * Don't mock the redux store, bypass it by testing on the unconnected view (See `this <https://hackernoon.com/unit-testing-redux-connected-components-692fa3c4441c>`_ for motivation)
    * Don't use shallow rendering (See `this <https://kentcdodds.com/blog/why-i-never-use-shallow-rendering>`_ for motivation)

Basic Pattern:

.. code:: jsx

    import React from "react";
    import { Component } from "./Component";
    import { render, fireEvent } from "@testing-library/react";

    // Necessary for ChildComponents that depend on being connected to the Redux store,
    // if you find you have a lot of these statements it may be an indication that you should
    // refactor them into Presentational Components and pass the state down from the parent
    // as props
    //
    // Alternatively these can be defined in views/__mocks__/ChildComponent.js
    jest.mock("./ChildComponent", () => ({
      __esModule: true,
      default: () => <div>Mocked Child Component</div>
    }));

    // Mock every action from `mapDispatchToProps` in the Component's connect call
    const action1Mock = jest.fn();
    const action2Mock = jest.fn();

    // Testing helper to include defaults that particular cases don't care about
    // and to hook up the mocks
    const renderComponent = function({
      prop1 = "prop1 default",
      prop2 = "prop2 default"
    }) {
      // Additional requirements for rendering can be handled here such as wrapping in a <Router>
      return render(
        <Component
          prop1={prop1}
          prop2={prop2}
          action1={action1Mock}
          action2={action2Mock}
        />
      );
    };

    const renderTest = options =>
      expect(renderComponent(options).asFragment()).toMatchSnapshot();

    // Enumerate various rendering cases based on the value of the Component's props
    // Snapshots are committed into source control and should be checked on code review to
    // make sure diffs correspond to appropriate Component changes
    test("Rendering for when prop1 is foo", () => renderTest({ prop1: "foo" }));
    test("Rendering for when prop2 is bar", () => renderTest({ prop1: "bar" }));
    // ...

    // Tests involving user interactions
    test("Clicking the CTA", () => {
      const component = renderComponent({});

      fireEvent.click(component.getByText("CTA"));
      expect(action1Mock.mock.calls).toEqual([[{ action: "foo" }]]);
    });

To avoid test order issues, set jest to automatically clear mocks after each test in ``package.json``:

.. code:: json

  "jest": {
    "clearMocks": true
  },

Presentational components can be tested in the same pattern. Each View should be tested but simpler components may only
need a couple cases:

.. code:: jsx

    import React from "react";
    import Simple from "./Simple";
    import { render } from "@testing-library/react";

    test("Render state when foo is on", () => {
      expect(render(<Simple foo={true} />).asFragment()).toMatchSnapshot();
    });

    test("Render state when foo is off", () => {
      expect(render(<Simple foo={false} />).asFragment()).toMatchSnapshot();
    });

Or just a smoke test:

.. code:: jsx

    import React from "react";
    import Simple from "./Simple";
    import { render } from "@testing-library/react";

    test("Basic rendering", () => {
      expect(() => render(<Simple />)).not.toThrow();
    });

.. note::

    One thing that isn't tested with this approach is the ``connect`` call itself. This function can be tested in isolation
    but ideally there shouldn't be much logic there and if there is it should probably be moved down into the selectors.

.. note::

    At the moment snapshot testing seems to me to be a good approach for asserting against rendered HTML, however it is
    somewhat controversial, see the following for further discussion:

    * `Jest Snapshot Testing <https://jestjs.io/docs/en/snapshot-testing>`_
    * `When to use Jest snapshot tests <https://codewithhugo.com/abusing-jest-snapshot-tests-some-nice-use-cases-/>`_
    * `Effective Snapshot Testing <https://kentcdodds.com/blog/effective-snapshot-testing>`_

Testing Routes
""""""""""""""
    * Given: a route
    * Expect: a component to be rendered with particular props

Create a mock for each Component rendered under the App's routes in ``views/__mocks__/Component.js``:

.. code:: jsx

    import React from "react"

    export const MockComponent = jest.fn();

    MockComponent.mockReturnValue(<div>Mocked Component</div>);

    const mock = jest.fn().mockImplementation(MockComponent);

    export default mock

Basic Pattern:

.. code:: jsx

    import React from "react";
    import { MemoryRouter} from "react-router-dom"
    import { render } from "@testing-library/react";
    import { MockComponent2 } from "./Component2";
    import App from "./App";

    jest.mock("./Component1");
    jest.mock("./Component2");

    const renderComponent = function(route) {
      return render(
        <MemoryRouter initialEntries={[route]}>
          <App />
        </MemoryRouter>
      );
    };

    test("Default route", () => {
      expect(renderComponent("/")).toMatchSnapshot();
    });

    test("Some route with params", () => {
      expect(renderComponent("/route/foo/bar")).toMatchSnapshot();
      expect(MockComponent2.mock.calls).toEqual([[{ p1: "foo", p2: "bar" }, {}]]);
    });

Testing Selectors
"""""""""""""""""
    * Given: a state
    * Expect: a projection
    * Define an initial state to simplify testing basic scenarios
    * Use `Immer`_ to produce readonly state objects to mimic what the selectors would actually receive

Basic Pattern:

.. code:: javascript

    import { selector1, selector2 } from "./index.js"
    import { produce } from "immer";

    const initialState = produce({}, () => ({
      foo: "bar",
      baz: true,
    }));

    test("selector1", () => {
      expect(selector1(initialState)).toBe("bar");
    });

    test("selector2 true", () => {
      expect(selector2(initialState)).toBe(true);
    });

    test("selector2 false", () => {
      const state = produce(initialState, draft => {
        draft.baz = false;
      });

      expect(selector2(initialState)).toBe(false);
    });

Testing Actions
"""""""""""""""
    * Given: a state and a requested state transition
    * Expect: a final state
    * Define an initial state to simplify testing basic scenarios
    * Use your own selectors to verify updated state

Basic Pattern:

.. code:: javascript

    import { rootReducer, action1, selector1 } from "./index.js"
    import { produce } from "immer";

    const initialState = produce({}, () => ({
      foo: "bar",
      baz: true,
    }));

    test("action1", () => {
      const updatedState = rootReducer(initialState, action1({ /* payload */ }));
      expect(selector1(updatedState)).toBe("bar after action is applied");
    });

Testing Async Operations
""""""""""""""""""""""""
    * Given: a state, a requested async operation, and a set of mocked API responses
    * Expect: a final state
    * Use `redux-saga-tester <https://github.com/wix/redux-saga-tester>`_
    * Actually `run the saga <https://dev.to/phil/the-best-way-to-test-redux-sagas-4hib>`_
    * `Mock the API <https://jestjs.io/docs/en/es6-class-mocks#calling-jestmockdocsenjest-objectjestmockmodulename-factory-options-with-the-module-factory-parameter>`_
    * Use selectors to verify state after the async operation is performed

.. code:: bash

    npm install --save-exact redux-saga-tester

Setup the API mock in ``api/__mocks__/index.js``:

.. code:: javascript

    export const mockApiMethod1 = jest.fn();
    export const mockApiMethod2 = jest.fn();
    // ...

    const mock = jest.fn().mockImplementation(() => ({
      apiMethod1: mockApiMethod1,
      apiMethod2: mockApiMethod2,
      // ...
    }));

    export default mock;

Basic Pattern

.. code:: javascript

    import { someAsyncRequest, rootReducer, rootSaga, selector1 } from "./index";
    import SagaTester from "redux-saga-tester";
    import { mockApiMethod1 } from "src/api";

    jest.mock("src/api");

    const startTest = (action, initialState) => {
      const sagaTester = new SagaTester({
        initialState,
        reducers: rootReducer
      });

      sagaTester.start(rootSaga);
      sagaTester.dispatch(action);

      return sagaTester;
    };

    test("someAsyncRequest success", async () => {
      mockApiMethod1.mockResolvedValue({ /* response */ });

      const sagaTest = startTest(someAsyncRequest({ /* payload */ }));

      await sagaTest.waitFor("someAsyncRequest/success");

      expect(selector1(sagaTest.getState())).toEqual(/* expectation */);
    });

Testing API Calls
"""""""""""""""""
    * Given: an API method invocation
    * Expect: a network call
    * Use `jest-fetch-mock <https://www.npmjs.com/package/jest-fetch-mock>`_

.. code:: bash

    npm install --save-exact jest-fetch-mock

Add to ``setupTests.js``:

.. code:: javascript

    global.fetch = require('jest-fetch-mock')

Basic pattern:

.. code:: javascript

    import API from "./index";

    const api = new API();

    test("Some fetch", async () => {

      // ...

      fetch.once(JSON.stringify({ /* response */ }));
      const result = await api.apiMethod(/* apiArgs */);

      expect(result).toEqual(/* expectedResult */);
      expect(fetch.mock.calls).toEqual([
        /* [expectedUrl, expectedFetchOptions] */
      ]);
    });

Code Style
----------

* Use `Prettier <https://prettier.io/>`_
* Use `ESLint <https://eslint.org/>`_
* Use `Husky <https://github.com/typicode/husky>`_
* Use `lint-staged <https://github.com/okonet/lint-staged>`_

.. code:: bash

    npm install --save-exact husky lint-staged prettier

Setup a pre-commit hook in ``package.json`` to automatically run Prettier and ESLint:

.. code:: json

  "husky": {
    "hooks": {
      "pre-commit": "lint-staged"
    }
  },
  "lint-staged": {
    "src/**/*.{css,scss,md}": [
      "prettier --write",
      "git add"
    ],
    "src/**/*.{js,jsx,json}": [
      "eslint",
      "prettier --write",
      "git add"
    ]
  },

This will only run Prettier on staged files going forward, so you should also do an initial Prettier run across all source files.
Add a ``pretty`` npm run command to ``package.json``:

.. code:: json

  "scripts": {

    "pretty": "prettier --write src/**/*.{js,jsx,json,css,scss,md}"

  },

And run it:

.. code:: bash

    npm run pretty

Adding Dependencies
-------------------

* Add new dependencies using ``npm install --save-exact <package>``

    * No point in using ``devDependencies`` if the package is not a library
    * Use ``--save-exact`` to avoid deployment bugs when a minor or patch dependency version update breaks compatibility


Deployment
----------

* Setup hosting for a :ref:`static-website-hosting`

Create a production build:

.. code:: bash

    npm run build

Copy over the build:

.. code:: bash

    rsync -avzr --delete build/ <server>:/var/www/<hostname>/html

CI/CD
-----

* Use `GitHub Actions <https://help.github.com/en/actions/automating-your-workflow-with-github-actions>`_

Add a lint npm run command to ``package.json``:

.. code:: json

  "scripts": {

    "lint": "eslint src/**/*.{js,json}",

  },

Generate a password-less SSH key and copy over the public key to ``.ssh/authorized_keys`` on the server being deployed to.
In the GitHub repo for the project add a DEPLOY_KEY secret and paste in the private key. Add a DEPLOY_DESTINATION secret
and paste in ``<username>@<server>:/var/www/<hostname>/html``

Create a ``.github/workflows/deploy.yml`` action:

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
        - name: Use Node.js 12.x
          uses: actions/setup-node@v1
          with:
            node-version: '12.x'
        - name: npm install, test, and build
          run: |
            npm ci
            npm run lint
            npm test
            npm run build
          env:
            CI: true
            name: CI
        - name: Deploy the build
          id: deploy
          uses: Pendect/action-rsyncer@v1.1.0
          env:
            DEPLOY_KEY: ${{secrets.DEPLOY_KEY}}
          with:
            flags: '-avzr --delete'
            options: ''
            ssh_options: ''
            src: 'build/'
            dest: ${{ secrets.DEPLOY_DESTINATION }}
        - name: Display status from deploy
          run: echo "${{ steps.deploy.outputs.status }}"

Push to the repo to trigger the action

TL;DR
-----

See `this repo <https://github.com/jpmunz/sample-web-app>`_ for an example project that encapsulates these tips.

References
----------

* `Describing UI state with data <http://nicolashery.com/describing-ui-state-with-data/>`_
* `Create React App <https://create-react-app.dev/docs/getting-started/>`_
* `React <https://reactjs.org/docs/getting-started.html>`_
* `Redux <https://redux.js.org/introduction/getting-started>`_
* `React Redux <https://react-redux.js.org/introduction/quick-start>`_
* `Redux Toolkit <https://redux-toolkit.js.org/introduction/quick-start>`_
* `Immer`_
* `React DOM Router <https://reacttraining.com/react-router/web/guides/quick-start>`_
* `Redux Saga <https://redux-saga.js.org/>`_
* `Mocking is a Code Smell <https://medium.com/javascript-scene/mocking-is-a-code-smell-944a70c90a6a>`_
* `That's Not Yours <https://8thlight.com/blog/eric-smith/2011/10/27/thats-not-yours.html>`_
* `Using Prettier and husky to make your commits safe <https://medium.com/@bartwijnants/using-prettier-and-husky-to-make-your-commits-save-2960f55cd351>`_

.. _Immer: https://github.com/immerjs/immer
