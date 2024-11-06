为 AnyIO 做出贡献
=====================

**Contributing to AnyIO**

.. tabs::

    .. tab:: 中文

      如果您希望为 AnyIO 提供修复或功能，请遵循以下指南。

      当您针对主 AnyIO 代码库发起拉取请求时，Github 会在您的修改代码上运行 AnyIO 测试套件。在发起拉取请求之前，您应该确保修改后的代码在本地通过了测试。为此，建议使用 tox_ 。默认的 tox 运行首先会执行 ``pre-commit``，然后运行实际的测试套件。要并行运行所有环境的检查，可以使用 ``tox -p`` 命令。

      要构建文档，请运行 ``tox -e docs``，这将生成一个名为 ``build`` 的目录，您可以在其中查看格式化后的 HTML 文档。

      AnyIO 使用 pre-commit_ 执行多个代码风格/质量检查。建议在您的本地代码库克隆中启用 pre-commit_ （使用 ``pre-commit install`` ），以确保您的更改在 GitHub 上也能通过相同的检查。

    .. tab:: 英文

      If you wish to contribute a fix or feature to AnyIO, please follow the following
      guidelines.

      When you make a pull request against the main AnyIO codebase, Github runs the AnyIO test
      suite against your modified code. Before making a pull request, you should ensure that
      the modified code passes tests locally. To that end, the use of tox_ is recommended. The
      default tox run first runs ``pre-commit`` and then the actual test suite. To run the
      checks on all environments in parallel, invoke tox with ``tox -p``.

      To build the documentation, run ``tox -e docs`` which will generate a directory named
      ``build`` in which you may view the formatted HTML documentation.

      AnyIO uses pre-commit_ to perform several code style/quality checks. It is recommended
      to activate pre-commit_ on your local clone of the repository (using
      ``pre-commit install``) to ensure that your changes will pass the same checks on GitHub.

.. _tox: https://tox.readthedocs.io/en/latest/install.html
.. _pre-commit: https://pre-commit.com/#installation

在 Github 上发起拉取请求
-------------------------------

**Making a pull request on Github**

.. tabs::

    .. tab:: 中文

      要将您的更改合并到主代码库，您需要一个 Github 账户。

      #. 通过导航到 `主 AnyIO 仓库 <main AnyIO repository>`_ 并点击右上角的 "Fork" 来分叉该仓库（如果您还没有自己的仓库副本）。
      #. 使用以下命令将分叉的仓库克隆到您的本地计算机： ``git clone git@github.com/yourusername/anyio``。
      #. 为您的拉取请求创建一个分支，例如： ``git checkout -b myfixname``。
      #. 对代码库进行所需的更改。
      #. 在本地提交您的更改。如果您的更改解决了一个现有问题，在提交信息中添加 ``Fixes XXX.`` 或 ``Closes XXX.``（其中 XXX 是问题编号）。
      #. 将更改集推送到您的分叉仓库（ ``git push`` ）。
      #. 导航到原始仓库的拉取请求页面（而不是您的分叉）并点击 "New pull request"。
      #. 点击 "compare across forks"。
      #. 选择您自己的分叉作为头仓库，然后选择正确的分支名称。
      #. 点击 "Create pull request"。

      如果遇到问题，请参考 `pull request 制作指南 <pull request making guide>`_ 以获取帮助。

    .. tab:: 英文

      To get your changes merged to the main codebase, you need a Github account.

      #. Fork the repository (if you don't have your own fork of it yet) by navigating to the
         `main AnyIO repository`_ and clicking on "Fork" near the top right corner.
      #. Clone the forked repository to your local machine with
         ``git clone git@github.com/yourusername/anyio``.
      #. Create a branch for your pull request, like ``git checkout -b myfixname``
      #. Make the desired changes to the code base.
      #. Commit your changes locally. If your changes close an existing issue, add the text
         ``Fixes XXX.`` or ``Closes XXX.`` to the commit message (where XXX is the issue
         number).
      #. Push the changeset(s) to your forked repository (``git push``)
      #. Navigate to Pull requests page on the original repository (not your fork) and click
         "New pull request"
      #. Click on the text "compare across forks".
      #. Select your own fork as the head repository and then select the correct branch name.
      #. Click on "Create pull request".

      If you have trouble, consult the `pull request making guide`_ on opensource.com.

.. _main AnyIO repository: https://github.com/agronholm/anyio
.. _pull request making guide:
    https://opensource.com/article/19/7/create-pull-request-github
