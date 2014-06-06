.. _tut-1:

Tutorial Part 1: Creating an add-on with a custom Look and Feel
===============================================================

Let's learn by example.  We'll create an add-on package that will:

- change the look and feel of Kotti by registering an additional CSS file
- add content types and forms

.. note::

    If you have questions going through this tutorial, please post
    a message to the `mailing list`_ or join the `#kotti`_ channel on
    irc.freenode.net to chat with other Kotti users who might be
    able to help.

In this part of the tutorial, we'll concentrate on how to create the
new add-on package, how to install and register it with our site, and how
to manage static resources in Kotti.

Kotti add-ons are proper Python packages. A number of them are available on
PyPI_. They include `kotti_media`_, for adding a set of video and audio content
types to a site, `kotti_gallery`_, for adding a photo album content type,
`kotti_blog`_, for blog and blog entry content types, etc.

The add-on we will make, kotti_mysite, will be just like those, in
that it will be a proper Python package created with the same command
line tools used to make `kotti_media`_, `kotti_blog`_, and the others.
We will set up kotti_mysite for our Kotti site, in the same way that
we might wish later to install, for example, `kotti_media`_.

So, we are working in the ``mysite`` directory, a virtualenv. We will
create the add-on as ``mysite/kotti_mysite``. kotti_mysite will be a
proper Python package, installable into our virtualenv.

.. _mailing list: http://groups.google.com/group/kotti
.. _#kotti: //irc.freenode.net/#kotti
.. _PyPI: http://pypi.python.org/pypi?%3Aaction=search&term=kotti_&submit=search/
.. _kotti_media: http://pypi.python.org/pypi/kotti_media/
.. _kotti_gallery: http://pypi.python.org/pypi/kotti_gallery/
.. _kotti_blog: http://pypi.python.org/pypi/kotti_blog/
.. _bobtemplates.kotti: http://pypi.python.org/pypi/bobtemplates.kotti/

Creating the Add-On Package
---------------------------

To create our add-on, we'll use a starter template from
``bobtemplates.kotti``.  For this, we'll need to first install the
``bobtemplates.kotti`` package into our virtualenv (the one that was created
during the :ref:`installation`).

.. code-block:: bash

  mkdir mysite && cd mysite && virtualenv .
  bin/pip install mr.bob
  bin/pip install bobtemplates.kotti==0.2a1

With ``bobtemplates.kotti`` installed, we can now create the skeleton for
the add-on package:

.. code-block:: bash

  bin/mrbob -O kotti_mysite bobtemplates.kotti:addon

Running this command, it will ask us a number of questions. The first question
asks for the name, what should be answered with ``kotti_mysite``. Hit
enter for the next questions to accept the defaults.  For the question ``Class
name for the content type`` enter ``Poll``, also as human readable name. We will
need this for the second part of our tutorial. For a full explanation of the questions
consult the README of `bobtemplates.kotti`_ itself.

When finished, observe that a new directory called ``kotti_mysite`` was added to the
current working directory, as mysite/kotti_mysite.

Installing Our New Add-On
-------------------------

To install the add-on (or any add-on, as discussed above) into our Kotti
site, we'll need to do two things:

- install the package into our virtualenv
- include the package inside our site's ``app.ini``

.. note::

  Why two steps?  Installation of our add-on as a Python package is
  different from activating the add-on in our site. Consider that you
  might have multiple add-ons installed in a virtualenv, but you could
  elect to activate a subset of them, as you experiment or develop add-ons.

To install the package into the virtualenv, we'll change into the new
``kotti_mysite`` directory, and issue a ``python setup.py develop``.
This will install the package in *development mode*:

.. code-block:: bash

  cd kotti_mysite
  ../bin/pip install -r https://raw.githubusercontent.com/Kotti/Kotti/0.10a2/requirements.txt
  ../bin/python setup.py develop
  wget https://raw.githubusercontent.com/Kotti/Kotti/0.10a2/app.ini

.. note::

  ``python setup.py install`` is for normal installation of a finished package,
  but here, for kotti_mysite, we will be developing it for some time, so we
  use ``python setup.py develop``. Using this mode, a special link file is
  created in the site-packages directory of your virtualenv. This link points
  to the add-on directory, so that any changes you make to the software will
  be reflected immediately without having to do an install again.

Step two is configuring our Kotti site to include our new
``kotti_mysite`` package.  To do this, open the ``app.ini`` file, which
you downloaded during :ref:`installation`.  Find the line that says:

.. code-block:: ini

  kotti.configurators = kotti_tinymce.kotti_configure

And add ``kotti_mysite.kotti_configure`` to it:

.. code-block:: ini

  kotti.configurators =
      kotti_tinymce.kotti_configure
      kotti_mysite.kotti_configure

Now you're ready to fire up the Kotti site again:

.. code-block:: bash

  cd ..
  bin/pserve app.ini


Adding static resources
-----------------------
Static resources are your CSS and Javascript files in the main. In the next
steps we will see how Kotti handles the static resources files and how these
files are included in the site.

Lets give the title a litte shadow. Take a look into the directory
``kotti_mysite/kotti_mysite/static/``. This is where the CSS file lives. Open
the the file ``kotti_mysite/kotti_mysite/static/style.css`` in your editor and
add a style to it:

.. code-block:: css

  h1, h2, h3 {
    text-shadow: 4px 4px 2px #ccc;
  }

How is it hooked up with Kotti?  Kotti uses fanstatic_ for managing
its static resources.  fanstatic_ has a number of cool features -- you
may want to check out their homepage to find out more.

Take a look at ``kotti_mysite/kotti_mysite/fanstatic.py`` to see how the
creation of the necessary fanstatic components is done:

.. code-block:: python

  from __future__ import absolute_import

  from fanstatic import Group
  from fanstatic import Library
  from fanstatic import Resource
  from js.jquery import jquery

  library = Library('kotti_mysite', 'static')

  css = Resource(
      library,
      'css/style.css',
      minified='css/style.min.css'
  )

  js = Resource(
      library,
      'js/script.js',
      minified='js/script.min.js',
      depends=[jquery, ]
  )

  kotti_mysite = Group([css, js, ])

To integrate your group into your setup add the following line to the
``kotti_configure`` function in the file
``kotti_mysite/kotti_mysite/__init__.py``:

.. code-block:: python

  def kotti_configure(settings):
      ...
      settings['kotti.fanstatic.view_needed'] += ' kotti_mysite.fanstatic.kotti_mysite'

Restart your kotti site, visit the site in your browser and notice the shadow
of the titles. A closer look to ``kotti.configurators`` you'll find in the next
section.

If you wanted to add a new JavaScript file, you would do this very
similarly. To add a JavaScript file called mysite_script.js, you would add a
fanstatic_ resource for it in ``kotti_mysite/kotti_mysite/fanstatic.py``
like so:

.. code-block:: python

  mysite_js = Resource(library, "mysite_script.js")

And change the last line to:

.. code-block:: python

  kotti_mysite = Group([css, js, mysite_js])

.. _fanstatic: http://www.fanstatic.org/

Configuring the Package with ``kotti.configurators``
----------------------------------------------------

Remember when we added ``kotti_mysite.kotti_configure`` to the
``kotti.configurators`` setting in the ``app.ini`` configuration file?
This is how we told Kotti to call additional code on start-up, so that
add-ons have a chance to configure themselves.  The function in
``kotti_mysite`` that is called on application start-up lives in
``kotti_mysite/kotti_mysite/__init__.py``.  Let's take a look:

.. code-block:: python

  def kotti_configure(settings):
      settings['kotti.fanstatic.view_needed'] += ' kotti_mysite.fanstatic.kotti_mysite'

Here, ``settings`` is a Python dictionary with all configuration variables in
the ``[app:kotti]`` section of our ``app.ini``, plus the defaults.  The values
of this dictionary are merely strings.  Notice how we add to the string
``kotti.fanstatic.view_needed``.

.. note::

   Note the initial space in ' kotti_mysite.static.kotti_mysite_group'. This
   allows a handy use of += on different lines. After concatenation of the
   string parts, blanks will delimit them.

This ``kotti.fanstatic.view_needed`` setting, in turn, controls which
resources are loaded in the public interface (as compared to the edit
interface).

As you might have guessed, we could have also completely replaced
Kotti's resources for the public interface by overriding the
``kotti.fanstatic.view_needed`` setting instead of adding to it, like
this:

.. code-block:: python

  def kotti_configure(settings):
      settings['kotti.fanstatic.view_needed'] = ' kotti_mysite.fanstatic.kotti_mysite'

This is useful if you've built your own custom theme.
Alternatively, you can completely :ref:`override the master template
<asset_overrides>` for even more control (e.g. if you don't want to
use Bootstrap).

See also :ref:`configuration` for a full list of Kotti's configuration
variables, and :ref:`static resources` for a more complete discussion
of how Kotti handles static resources through fanstatic.

In the :ref:`next part <tut-2>` of the tutorial, we'll add our first
content types, and add forms for them.
