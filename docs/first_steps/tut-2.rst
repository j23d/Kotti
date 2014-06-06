.. _tut-2:

Tutorial Part 2: A Content Type
===============================

Kotti's default content types include ``Document``, ``Image`` and ``File``.  In
this part of the tutorial, we'll add add to these built-in content types by
making a ``Poll`` content type which will allow visitors to view polls and vote
on them.

Adding Models
-------------

Do you remember the question for the name of your content type in the
:ref:`last part <tut-1>` of the tutorial? We answered with ``Poll``. The
bob template prepared the first of our content types in the file at
``kotti_mysite/kotti_mysite/resources.py``. Open it and lets look to the
definition of the ``Poll`` content type:

.. code-block:: python

    from kotti.resources import Content
    from sqlalchemy import Column
    from sqlalchemy import ForeignKey
    from sqlalchemy import Integer
    from sqlalchemy import Unicode

    from kotti_mysite import _


    class Poll(Content):
        """My content type"""

        id = Column(
            Integer(),
            ForeignKey('contents.id'),
            primary_key=True
        )

        # Add additional columns here
        example_attribute = Column(
            Unicode()
        )

        type_info = Content.type_info.copy(
            name=u'Poll',
            title=_(u'Poll'),
            add_view=u'add_poll',
            addable_to=['Document', ],
            selectable_default_views=[
                ('alternative-view', _(u"Alternative View")),
            ],
        )

        def __init__(self, example_attribute=u"", **kwargs):
            super(Poll, self).__init__(**kwargs)
            self.example_attribute = example_attribute


Things to note here:

- Kotti's content types use SQLAlchemy_ for definition of persistence.

- ``Poll`` derives from :class:`kotti.resources.Content`, which is the
  common base class for all content types.

- ``Poll`` declares a sqla.Column ``id``, which is required to hook
  it up with SQLAlchemy's inheritance.

- The type_info class attribute does essential configuration. We
  refer to name and title, two properties already defined as part of
  ``Content``, our base class.  The ``add_view`` defines the name of the add
  view, which we'll come to in a second.  The ``addable_to`` defines which
  content types we can add ``Poll`` items to. Finally,
  ``selectable_default_views`` defines an array of alternative views for
  the content type.

- We do not need to define any additional sqlaColumn() properties, as the title
  is the only property we need for this content type. However the scaffold shows
  us how to create new attributes for a content type. See this in the lines with
  the name ``example_attribute`` in.

We'll add another content class to hold the choices for the poll.  Add
this into the same ``resources.py`` file:

.. code-block:: python

  class Choice(Content):
      id = sqla.Column(
          Integer(), ForeignKey('contents.id'), primary_key=True)
      votes = Column(Integer())

      type_info = Content.type_info.copy(
          name=u'Choice',
          title=u'Choice',
          add_view=u'add_choice',
          addable_to=[u'Poll'],
          )

      def __init__(self, votes=0, **kwargs):
          super(Choice, self).__init__(**kwargs)
          self.votes = votes

The ``Choice`` class looks very similar to ``Poll``.  Notable
differences are:

- It has an additional sqla.Column property called ``votes``.  We'll use this
  to store how many votes were given for the particular choice.  We'll again
  use the inherited ``title`` column to store the title of our choice.

- The ``type_info`` defines the title, the ``add_view`` view, and that
  choices may only be added *into* ``Poll`` items, with the line
  ``addable_to=[u'Poll']``.

Adding Forms and a View
-----------------------

Views (including forms) are typically put into a module called
``views``.  Let's open the file in this module at
``kotti_mysite/kotti_mysite/views.py`` and look at the following code:

.. code-block:: python

  from colander import SchemaNode
  from colander import String
  from kotti.views.edit.content import ContentSchema

  from kotti_mysite import _

  class PollSchema(ContentSchema):
      """Schema for add / edit forms of Poll"""

      example_attribute = SchemaNode(
          String(),
          title=_(u'Example Attribute'),
          missing=u"",
      )

This is the definition for our Poll. You see that the schema is derived from
the ``ContentSchema`` of Kotti itself, what provides a few basic attributes
like title, description and body. Lets now add the schema for the choices:

.. code-block:: python

  class ChoiceSchema(ContentSchema):
      pass

Colander_ is the library that we use to define our schemas.  Colander
allows us to validate schemas against form data.

The two classes define the schemas for our add and edit forms.  That
is, they specify which fields we want to display in the forms.

In the file ``views.py`` we already have definitions for the polls:

.. code-block:: python

    from kotti.views.form import AddFormView
    from kotti.views.form import EditFormView
    from pyramid.view import view_config
    from pyramid.view import view_defaults

    @view_config(name=Poll.type_info.add_view,
                 permission='add',
                 renderer='kotti:templates/edit/node.pt')
    class PollAddForm(AddFormView):

        schema_factory = PollSchema
        add = Poll
        item_type = _(u"Poll")


    @view_config(name='edit',
                 context=Poll,
                 permission='edit',
                 renderer='kotti:templates/edit/node.pt')
    class PollEditForm(EditFormView):

        schema_factory = PollSchema

We are using the `view_config`_ decorator from pyramid to adding the view
configuration to our site.


Let's move on to building the form for the choices.  Add this to ``views.py``:

.. code-block:: python

    from kotti_mysite.resources import Choice

    @view_config(name=Choice.type_info.add_view,
                 permission='add',
                 renderer='kotti:templates/edit/node.pt')
    class ChoiceAddForm(AddFormView):

        schema_factory = ChoiceSchema
        add = Choice
        item_type = _(u"Choice")


    @view_config(name='edit',
                 context=Choice,
                 permission='edit',
                 renderer='kotti:templates/edit/node.pt')
    class ChoiceEditForm(EditFormView):

        schema_factory = ChoiceSchema


Using the ``AddFormView`` and ``EditFormView`` base classes from
Kotti, these forms are simple to define. We associate the schemas
defined above, setting them as the schema_factory for each form,
and we specify the content types to be added by each.

Wiring up the Content Types and Forms
-------------------------------------

It's time for us to see things in action. For that, some configuration
of the types and forms is in order.

Find ``kotti_mysite/kotti_mysite/__init__.py`` and add configuration that
registers our new code in the Kotti site.

We change the ``kotti_configure`` function to look like:

.. code-block:: python

  def kotti_configure(settings):
      settings['kotti.fanstatic.view_needed'] += (
          ' kotti_mysite.fanstatic.kotti_mysite_group')
      settings['kotti.available_types'] += (
          ' kotti_mysite.resources.Poll kotti_mysite.resources.Choice')
      settings['pyramid.includes'] += ' kotti_mysite'

Here, we've added our two content types to the site's available_types, a global
registry.

Because we are using the `view_config`_ decorator our add and edit forms are
added automatically to the configuration with one call in the file
``__init__.py``::

.. code-block:: python

    def includeme(config):

        config.add_translation_dirs('kotti_mysite:locale')
        config.scan(__name__)

With the call to ``config.scan(__name__)`` the whole package is searched for
decorated classes and functions and its added to the coniguration. We take
a deeper look to the possible configuration options:


.. code-block:: python

    @view_config(name=Poll.type_info.add_view,
                 permission='add',
                 renderer='kotti:templates/edit/node.pt')
    class PollAddForm(AddFormView):

        schema_factory = PollSchema
        add = Poll
        item_type = _(u"Poll")


    @view_config(name='edit',
                 context=Poll,
                 permission='edit',
                 renderer='kotti:templates/edit/node.pt')
    class PollEditForm(EditFormView):

        schema_factory = PollSchema


    @view_config(name=Choice.type_info.add_view,
                 permission='add',
                 renderer='kotti:templates/edit/node.pt')
    class ChoiceAddForm(AddFormView):

        schema_factory = ChoiceSchema
        add = Choice
        item_type = _(u"Choice")


    @view_config(name='edit',
                 context=Choice,
                 permission='edit',
                 renderer='kotti:templates/edit/node.pt')
    class ChoiceEditForm(EditFormView):

        schema_factory = ChoiceSchema


The first argument of the decoration is the name of the view gives the name of
the view. We get the names of each add view, `add_poll` and `add_choice`, from
the set in the type_info class attribute of the types (Compare to the
classes where Poll() and Choice() are defined). The names of the edit views
are simply `edit`, the names of add views are simply `add`. We can, of course,
add our own view names, but `add` and `edit` should be used for adding and
editing respectively, as Kotti uses those names for its base functionality.


Adding a Poll and Choices to the site
-------------------------------------

Let's try adding a Poll and some choices to the site. Start the site up with
the command

.. code-block:: bash

  bin/pserve app.ini

Login with the username *admin* and password *qwerty* and click on the Add menu
button. You should see a few choices, namely the base Kotti classes
``Document``, ``File`` and ``Image`` and the Content Type we added, ``Poll``.

Lets go ahead and click on ``Poll``. For the question, let's write
*What is your favourite color?*. Now let's add three choices,
*Red*, *Green* and *Blue* in the same way we added the poll.

If we now go to the poll we added, we can see the question, but not our
choices, which is definitely not what we wanted. Let us fix this, shall we?

Adding a custom View to the Poll
--------------------------------

Since there are plenty tutorials on how to write TAL templates, we will not
write a complete one here, but just a basic one, to show off the general idea.

First, we need to write a view that will send the needed data (in our case,
the choices we added to our poll). Here is the code, added to ``views.py``.

.. code-block:: python

    from kotti_mysite.fanstatic import kotti_mysite

    @view_defaults(context=Poll, permission='view')
    class PollView(object):
        """View(s) for Poll"""

        def __init__(self, context, request):

            self.context = context
            self.request = request

        @view_config(name='view',
                     renderer='kotti_mysite:templates/poll.pt')
        def view(self):
            kotti_mysite.need()
            choices = context.values()
            return {
                'choices': choices
            }

To find out if a Choice was added to the ``Poll`` we are currently viewing, we
compare it's *parent_id* attribute with the *id* of the Poll - if they are the
same, the ``Choice`` is a child of the ``Poll``.
To get all the appropriate choices, we do a simple database query, filtered as
specified above.
Finally, we return a dictionary of all choices under the keyword *choices*.

Next on, we need a template to actually show our data. It could look something
like this. Look into the folder named ``templates`` and open the file
``poll.pt`` and change the content like this:

>>> TODO: Poll.pt have to be renamed to poll.pt in bobtemplates.kotti!

.. code-block:: html

  <!DOCTYPE html>
  <html xmlns:tal="http://xml.zope.org/namespaces/tal"
        xmlns:metal="http://xml.zope.org/namespaces/metal"
        metal:use-macro="api.macro('kotti:templates/view/master.pt')">

    <article metal:fill-slot="content" class="poll-view content">
      <h1>${context.title}</h1>
      <ul>
          <li tal:repeat="choice choices">
            <a href="${request.resource_url(choice)}">${choice.title}</a>
          </li>
      </ul>
    </article>

  </html>

The first 6 lines are needed so our template plays nicely with the master
template (so we keep the add/edit bar, base site structure etc.).
The next line prints out the context.title (our question) inside the <h1> tag
and then prints all choices (with links to the choice) as an unordered list.

The linking of the view definitions with the configuration are already done
with ``config.scan(__name__)`` in the ``inludeme`` function in ``__init__.py``.

With this, we are done with the second tutorial. Restart the server instance,
take a look at the new ``Poll`` view and play around with the template until
you are completely satisfied with how our data is presented.
If you will work with templates for a while (or anytime you're developing
basically) I'd recommend you use the pyramid *reload_templates* and
*debug_templates* options as they save you a lot of time lost on server
restarts.

.. code-block:: ini

  pyramid.reload_templates = true
  pyramid.debug_templates = true

In the :ref:`next tutorial <tut-3>`, we will learn how to enable our users to
actually vote for one of the ``Poll`` options.

.. _SQLAlchemy: http://www.sqlalchemy.org/
.. _Colander: http://colander.readthedocs.org/
.. _view_config: http://docs.pylonsproject.org/docs/pyramid/en/latest/narr/viewconfig.html#mapping-views-using-a-decorator-section

