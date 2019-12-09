Scripts
=======

* Omit the extension and specify the type in the file with a ``#!`` so the caller doesn't need to know the language of the script

To make it runnable:

.. code:: bash

    chmod a+x script
    mv script ~/bin # or sudo mv script /usr/local/bin

Bash
----

* Use if the command is simple
* Or, if you need to pipe through a lot of other commands

As a standalone ``script``:

.. code:: bash

    #!/bin/bash

    if [[ $# -eq 0 ]] ; then
        echo 'Usage: ...'
        exit 1
    fi

    echo $1 | foo | bar

As an inline function in ``.bashrc``:

.. code:: bash

    my-function ()
    {
        echo "$1" | ...
    }

Python
------

* Use if the command needs to do a lot of manipulation of data structures
* Use if the command requires a lot of argument parsing
* Use `argparse <https://docs.python.org/3/howto/argparse.html>`_


.. code:: python

    #!/usr/bin/env python3
    # -*- coding: utf-8 -*-

    import argparse


    def perform_op(arg1, flag1):
        # ...

    if __name__ == "__main__":
        parser = argparse.ArgumentParser(description='Some script')
        parser.add_argument("arg1")
        parser.add_argument("--flag1", action="store_true")
        args = parser.parse_args()
        perform_op(args.arg1, args.flag1)
