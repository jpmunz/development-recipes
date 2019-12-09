Documentation
========================

* Use `Sphinx <http://www.sphinx-doc.org/en/master/>`_
* Use `reStructuredText <https://docutils.sourceforge.io/rst.html>`_
* Don't use Markdown (see `this <https://www.ericholscher.com/blog/2016/mar/15/dont-use-markdown-for-technical-docs/>`_ for motivation)

Setup
-----

.. code:: bash

    cd <project>
    python3 -m venv venv
    . venv/bin/activate
    pip install sphinx sphinx-rtd-theme
    mkdir docs
    cd docs
    sphinx-quickstart

Edit ``docs/conf.py``:

.. code:: python

    import sphinx_rtd_theme

    extensions = [
        "sphinx_rtd_theme",
    ]

    html_theme = 'sphinx_rtd_theme'

Build
-----

.. code:: bash

    <project>/venv/bin/sphinx-build -b html <project>/docs <project>/docs/_build/html

To enable `Live Edit in WebStorm <https://www.jetbrains.com/help/webstorm/live-editing.html>`_ setup a
`File Watcher <https://www.jetbrains.com/help/webstorm/settings-tools-file-watchers.html>`_ using the above command and
run Debug on the generated html file in ``<project>/docs/html/`` for whichever file you're working on.

Deploy
------

Push to a public repo and host on `Read the Docs <https://docs.readthedocs.io/en/stable/intro/import-guide.html>`_.

References
----------

* `reStructuredText Primer <http://www.sphinx-doc.org/en/master/usage/restructuredtext/basics.html>`_
* `Sphinx Directives <http://www.sphinx-doc.org/en/master/usage/restructuredtext/directives.html>`_
*  `Why You Shouldn’t Use “Markdown” for Documentation <https://www.ericholscher.com/blog/2016/mar/15/dont-use-markdown-for-technical-docs/>`_
