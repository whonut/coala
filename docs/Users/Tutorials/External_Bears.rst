External Bears
==============

Welcome. This tutorial hopes to teach you how to use the
``@external_bear_wrap`` decorator in order to write Bears in languages other
than python.

.. note::

  This tutorial assumes that you already know the basics. If you are new please
  refer to
  :doc:`this link<Writing_Bears>`.

  If you are planning to create a bear that uses an already existing tool (aka
  linter), please refer to
  :doc:`this link instead<Linter_Bears>`.

Why is This Useful?
-------------------

coala is a great tool if you want to write static code analysis without having
to worry about the interaction with the user. That makes it an excellent choice
for researchers and other people who want to build new static analysis routines.
Enabling developers to write external Bears means that they do not need to waste
any time learning new languages and they can get faster to the interesting part,
**static code analysis**.

How Does This Work?
-------------------

By using the ``@external_bear_wrap`` decorator you will have all the necessary
data sent to your external executable (filename, lines, settings) as a JSON
string via stdin. Afterwards the analysis takes place in your executable that
can be written in **literally** any language. In the end you will have to
provide the ``Results`` in a JSON string via stdout.

In this tutorial we will go through 2 examples where we create a very simple
bear. The first example will use a **compiled** language that creates a
standalone binary, ``C++``, whilst in the second example we will take a look at
``JS`` that needs ``node`` in order to run out of the browser.

External Bear Generation Tool
-----------------------------

If you really do not want to write any python code, there is a tool
`here <https://gitlab.com/coala/coala-bear-management>`__,
``coala-bears-create``, that will create the wrapper for you. We will be using

::

    $ coala-bears-create -ext

in order to generate the wrapper for the bear.

Writing a Bear in C++
---------------------

In order to create the wrapper for our ``C++`` bear we just have to run
``coala-bears-create`` as mentioned above and answer the questions. We will
have to create a fresh directory to work with. The bear that we are building
will check whether there is any **coala** spelled with a capital ``C`` since
that is a horrible mistake for one to make. For that reason we will name the
Bear ``coalaCheckBear`` and the executable ``coalaCheck_cpp``. After the script
is finished running you should have 2 files in your selected directory:
``coalaCheckBear.py`` and ``coalaCheckBearTest.py``. We will focus on the first
for now, it should look something like this (after some cleaning):

::

    import os

    from coalib.bearlib.abstractions.ExternalBearWrap import external_bear_wrap


    @external_bear_wrap(executable='coalaCheck_cpp',
                        settings={})
    class coalaCheckBear:
        """
        Checks for coala written with uppercase 'C'
        """
        LANGUAGES = {'All'}
        REQUIREMENTS = {'coalaCheck_cpp'}
        AUTHORS = {'Me'}
        AUTHORS_EMAILS = {'me@mail.com'}
        LICENSE = 'GPLv3'

        @staticmethod
        def create_arguments():
            return ()

We will not be modifying the ``create_arguments`` method for this example. Now
we need to get our input. Since our input will be a JSON string we need some
kind of JSON class. `This one <https://github.com/nlohmann/json>`__ is a great
choice because it is easy to integrate and is used in this tutorial.

Let's create ``coalaCheck.cpp`` and start writing the first pieces of code. The
best thing about nlohmann's JSON library is that you can parse JSON directly
from stdin like this:

::

    #include <iostream>

    #include "json.hpp"

    using json = nlohmann::json;
    using namespace std;

    json in;

    int main() {

        cin >> in;

        cout << in;

    return 0;

We can test that by giving it a sample JSON string. Before testing we need to
build it though. The JSON library requires C++11 so a sample ``Makefile`` would
look like this:

::

    build: coalaCheck.cpp
        g++ -std=c++11 -o coalaCheck_cpp coalaCheck.cpp

We test our binary and it work but we don't quite do any code analysis yet.
We need to use the input we get. First of all we have to know the JSON spec that
is fed to us via stdin (`The JSON Spec`_). The filename is found in
``in["filename"]`` and the list of lines is found in ``in["file"]``. Let's make
a result adding function, also an init function proves quite useful for
initializing the output json.

::

    #include <iostream>
    #include <string>

    #include "json.hpp"

    using json = nlohmann::json;
    using namespace std;

    json in;
    json out;
    string origin;

    void init_results(string bear_name) {
        origin = bear_name;
        out["results"] = json::array({});
    }

    void add_result(string message, int line, int column, int severity) {
        json result = {
            {"origin", origin},
            {"message", message},
                {"affected_code", json::array({{
                    {"file", in["filename"]},
                    {"start", {
                        {"column", column},
                        {"file", in["filename"]},
                        {"line", line}
                    }},
                    {"end", {
                        {"column", column+6},
                        {"file", in["filename"]},
                        {"line", line}
                    }}
                }})},
            {"severity", severity}
        };
        out["results"] += result;
    }

    int main() {

        cin >> in;

        init_results("coalaCheckBear");

        cout << out;
        return 0;
    }

The ``C++`` operators and syntax are not well suited for JSON manipulation but
nlohmann's JSON lib makes it as easy as possible. The last thing we have to do
is to iterate over the lines and check for ``"coala"`` with an uppercase ``"C"``
. We can do that by using ``string``'s ``find`` function like so:

::

    #include <iostream>
    #include <string>

    #include "json.hpp"

    using json = nlohmann::json;
    using namespace std;

    json in;
    json out;
    string origin;

    void init_results(string bear_name) {
        origin = bear_name;
        out["results"] = json::array({});
    }

    void add_result(string message, int line, int column, int severity) {
        json result = {
            {"origin", origin},
            {"message", message},
                {"affected_code", json::array({{
                    {"file", in["filename"]},
                    {"start", {
                        {"column", column},
                        {"file", in["filename"]},
                        {"line", line}
                    }},
                    {"end", {
                        {"column", column+6},
                        {"file", in["filename"]},
                        {"line", line}
                    }}
                }})},
            {"severity", severity}
        };
        out["results"] += result;
    }

    int main() {

        cin >> in;

        init_results("coalaCheckBear");

        int i = 0;
        for (auto it=in["file"].begin(); it !=in["file"].end(); it++) {
            i++;
            string line = *it;
            size_t found = line.find("Coala");
            while (found != string::npos) {
                add_result("Did you mean 'coala'?", i, found, 2);
                found = line.find("Coala", found+1);
            }
        }

        cout << out;

        return 0;
    }

After building our executable we need to add it to the ``PATH`` env variable. We
could also modify the wrapper and give there the full path. We can add the
current directory to the ``PATH`` like so:

::

    $ export PATH=$PATH:$PWD

The last step is to test if everything is working properly. For that, you can
the testfile found in
`this repo <https://github.com/Redridge/coalaCheckBear-cpp>`__ which also
contains our final result. We can now see our Bear in action by running:

::

    $ coala -d . -b coalaCheckBear -f testfile

.. note::

  If you have ran ``coala`` over a file more than once without modifying it,
  coala will try to cache it. In order to avoid such behavior add
  ``--flush-cache`` at the end of the command.

Writing a Bear With Javascript and Node
---------------------------------------

What do we do when our source code needs some other binary in order to run?
Let's try writing a bear with out of browser Javascript powered on Node to see
how that works.

Firstly, we run ``coala-bears-create -ext`` but we supply ``node`` as the
executable name.

.. note::

  This tutorial uses ``node v6.2.2``. It should work with older versions too but
  we suggest that you update.

After the wrapper generation we will have to do a bit of editing of the
``create_arguments`` method. In particular, we have to add our source code file
as an argument (so that the command becomes ``node coalaCheck.js``). The
``create_arguments`` method returns a tuple so if we want to add only one
argument then we have to use a comma at the end (ex ``(one_item,)``).

::

    import os

    from coalib.bearlib.abstractions.ExternalBearWrap import external_bear_wrap


    @external_bear_wrap(executable='node',
                        settings={})
    class coalaCheckBear:
        """
        Checks for coala written with uppercase 'C'
        """
        LANGUAGES = {'All'}
        REQUIREMENTS = {'node'}
        AUTHORS = {'Me'}
        AUTHORS_EMAILS = {'me@mail.com'}
        LICENSE = 'GPLv3'

        @staticmethod
        def create_arguments():
            return ('coalaCheck.js',)

Now on to writing ``coalaCheck.js``. First we will add our I/O handling.

::

    var input = "";

    console.log = (msg) => {
        process.stdout.write(`${msg}\n`);
    };

    process.stdin.setEncoding('utf8');

    process.stdin.on('readable', () => {
        var chunk = process.stdin.read();
        if (chunk !== null) {
            input += chunk;
        }
    });

    process.stdin.on('end', () => {
        input = JSON.parse(input);
        console.log(JSON.stringify(input));
    });

We can now check if our I/O works by running ``node coalaCheck.js`` and
supplying a valid JSON string in the stdin. Next up we will add the init and the
add result functions.

::

    var out = {};
    var origin;

    init_results = (bear_name) => {
        origin = bear_name;
        out["results"] = [];
    };

    add_result = (message, line, column, severity) => {
        var result = {
            "origin": origin,
            "message": message,
            "affected_code": [{
                    "file": input["filename"],
                    "start": {
                        "column": column,
                        "file": input["filename"],
                        "line": line
                    },
                    "end": {
                        "column": column+6,
                        "file": input["filename"],
                        "line": line
                    }
                }],
            "severity": severity
        };
        out["results"].push(result)
    };

The last part is to iterate over the lines and check for ``"coala"`` spelled
with a capital ``"C"``. The final source should look like this:

::

    var input = "";
    var out = {};
    var origin;

    console.log = (msg) => {
        process.stdout.write(`${msg}\n`);
    };

    init_results = (bear_name) => {
        origin = bear_name;
        out["results"] = [];
    };

    add_result = (message, line, column, severity) => {
        var result = {
            "origin": origin,
            "message": message,
            "affected_code": [{
                    "file": input["filename"],
                    "start": {
                        "column": column,
                        "file": input["filename"],
                        "line": line
                    },
                    "end": {
                        "column": column+6,
                        "file": input["filename"],
                        "line": line
                    }
                }],
            "severity": severity
        };
        out["results"].push(result)
    };

    process.stdin.setEncoding('utf8');

    process.stdin.on('readable', () => {
        var chunk = process.stdin.read();
        if (chunk !== null) {
            input += chunk;
        }
    });

    process.stdin.on('end', () => {
        input = JSON.parse(input);
        init_results("coalaCheckBear");
        for (i in input["file"]) {
            var line = input["file"][i];
            var found = line.indexOf("Coala");
            while (found != -1) {
                add_result("Did you mean 'coala'?", parseInt(i)+1, found+1, 2);
                found = line.indexOf("Coala", found+1)
            }
        }
        console.log(JSON.stringify(out));
    });

And we are pretty much done. In order to run this Bear we do not need to add the
source code to the path because the binary being run is ``node``. Although there
is a problem: the argument we supplied will be looked up only in the current
directory. To fix this we can add the full path of the ``.js`` file in the
argument list. In this case we just run the bear from the same directory as
``coalaCheck.js``. The code for this example can be found
`here <https://github.com/Redridge/coalaCheckBear-js>`__.

The JSON Spec
-------------

coala will send you data in a JSON string via stdin and the executable has to
provide a JSON string via stdout. The specs are the following:

* input JSON spec

+--------------------------------+-------+-----------------------------------+
|Tree                            |Type   |Description                        |
+--------------------------------+-------+-----------------------------------+
|filename                        |str    |the name of the file being analysed|
+--------------------------------+-------+-----------------------------------+
|file                            |list   |file contents as a list of files   |
+--------------------------------+-------+-----------------------------------+
|settings                        |obj    |settings as key:value pairs        |
+--------------------------------+-------+-----------------------------------+

* output JSON spec

+--------------------------------+-------+-----------------------------------+
|Tree                            |Type   |Description                        |
+--------------------------------+-------+-----------------------------------+
|results                         |list   |list of results                    |
+---+----------------------------+-------+-----------------------------------+
|   |origin                      |str    |usually the name of the bear       |
+---+----------------------------+-------+-----------------------------------+
|   |message                     |str    |message to be displayed to the user|
+---+----------------------------+-------+-----------------------------------+
|   |affected_code               |list   |contains SourceRange objects       |
+---+---+------------------------+-------+-----------------------------------+
|   |   |file                    |str    |the name of the file               |
+---+---+------------------------+-------+-----------------------------------+
|   |   |start                   |obj    |start position of affected code    |
+---+---+---+--------------------+-------+-----------------------------------+
|   |   |   |file                |str    |the name of the file               |
+---+---+---+--------------------+-------+-----------------------------------+
|   |   |   |line                |int    |line number                        |
+---+---+---+--------------------+-------+-----------------------------------+
|   |   |   |column              |int    |column number                      |
+---+---+---+--------------------+-------+-----------------------------------+
|   |   |end                     |obj    |end position of affected code      |
+---+---+---+--------------------+-------+-----------------------------------+
|   |   |   |file                |str    |the name of the file               |
+---+---+---+--------------------+-------+-----------------------------------+
|   |   |   |line                |int    |line number                        |
+---+---+---+--------------------+-------+-----------------------------------+
|   |   |   |column              |int    |column number                      |
+---+---+---+--------------------+-------+-----------------------------------+
|   |severity                    |int    |severity of the result (0-2)       |
+---+----------------------------+-------+-----------------------------------+
|   |debug_msg                   |str    |message to be shown in DEBUG log   |
+---+----------------------------+-------+-----------------------------------+
|   |additional_info             |str    |additional info to be displayed    |
+---+----------------------------+-------+-----------------------------------+

.. note::

  The output JSON spec is the same as the one that ``coala-json`` uses. If you
  ever get lost you can run ``coala-json`` over a file and check the results.
