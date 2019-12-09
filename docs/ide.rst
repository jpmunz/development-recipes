IDE
===

Back-end Development
--------------------

    * Use `tmux <https://github.com/tmux/tmux/wiki>`_
    * Use `vim8 <https://www.vim.org/>`_ ...or `whatever <https://simpleprogrammer.com/text-editor-wars/>`_
    * See `this <https://lucasfcosta.com/2019/02/10/terminal-guide-2019.html>`_ for some justifications for this approach

Version control a collection of dotfiles to easily configure common tools on a new machine similar to `this repo <https://github.com/jpmunz/devenv>`_:

.. code:: bash

    git clone devenv
    cd devenv
    ./install-dotfiles.sh
    sudo ./install-packages.sh # if you have sudo access to the environment

Front-end Development
---------------------

    * Use `WebStorm <https://www.jetbrains.com/webstorm/>`_
    * Setup `Vim Emulation <https://www.jetbrains.com/help/webstorm/vim-emulation.html>`_
    * Use `create-react-app through WebStorm <https://www.jetbrains.com/help/webstorm/react.html>`_ to pre-configure linting, testing, Hot Module Replacement, etc.
    * Install the `React Developer Tools browser extension <https://chrome.google.com/webstore/detail/react-developer-tools/fmkadmapgofadopljbjfkapdkoienihi>`_
    * Install the `Redux DevTools browser extension <https://chrome.google.com/webstore/detail/redux-devtools/lmhkpmbekcpmknklioeibfkpmmfibljd>`_
