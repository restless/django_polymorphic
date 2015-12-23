.. _advanced-features:

Advanced features
=================

In the examples below, these models are being used::

    from django.db import models
    from polymorphic.models import PolymorphicModel

    class ModelA(PolymorphicModel):
        field1 = models.CharField(max_length=10)

    class ModelB(ModelA):
        field2 = models.CharField(max_length=10)

    class ModelC(ModelB):
        field3 = models.CharField(max_length=10)

Filtering for classes (equivalent to python's isinstance() ):
-------------------------------------------------------------

>>> ModelA.objects.instance_of(ModelB)
.
[ <ModelB: id 2, field1 (CharField), field2 (CharField)>,
  <ModelC: id 3, field1 (CharField), field2 (CharField), field3 (CharField)> ]

In general, including or excluding parts of the inheritance tree::

    ModelA.objects.instance_of(ModelB [, ModelC ...])
    ModelA.objects.not_instance_of(ModelB [, ModelC ...])

You can also use this feature in Q-objects (with the same result as above):

>>> ModelA.objects.filter( Q(instance_of=ModelB) )


Polymorphic filtering (for fields in inherited classes)
-------------------------------------------------------

For example, cherrypicking objects from multiple derived classes
anywhere in the inheritance tree, using Q objects (with the
syntax: ``exact model name + three _ + field name``):

>>> ModelA.objects.filter(  Q(ModelB___field2 = 'B2') | Q(ModelC___field3 = 'C3')  )
.
[ <ModelB: id 2, field1 (CharField), field2 (CharField)>,
  <ModelC: id 3, field1 (CharField), field2 (CharField), field3 (CharField)> ]


Combining Querysets
-------------------

Querysets could now be regarded as object containers that allow the
aggregation of different object types, very similar to python
lists - as long as the objects are accessed through the manager of
a common base class:

>>> Base.objects.instance_of(ModelX) | Base.objects.instance_of(ModelY)
.
[ <ModelX: id 1, field_x (CharField)>,
  <ModelY: id 2, field_y (CharField)> ]


ManyToManyField, ForeignKey, OneToOneField
------------------------------------------

Relationship fields referring to polymorphic models work as
expected: like polymorphic querysets they now always return the
referred objects with the same type/class these were created and
saved as.

E.g., if in your model you define::

    field1 = OneToOneField(ModelA)

then field1 may now also refer to objects of type ``ModelB`` or ``ModelC``.

A ManyToManyField example::

    # The model holding the relation may be any kind of model, polymorphic or not
    class RelatingModel(models.Model):
        many2many = models.ManyToManyField('ModelA')  # ManyToMany relation to a polymorphic model

    >>> o=RelatingModel.objects.create()
    >>> o.many2many.add(ModelA.objects.get(id=1))
    >>> o.many2many.add(ModelB.objects.get(id=2))
    >>> o.many2many.add(ModelC.objects.get(id=3))

    >>> o.many2many.all()
    [ <ModelA: id 1, field1 (CharField)>,
      <ModelB: id 2, field1 (CharField), field2 (CharField)>,
      <ModelC: id 3, field1 (CharField), field2 (CharField), field3 (CharField)> ]


Using Third Party Models (without modifying them)
-------------------------------------------------

Third party models can be used as polymorphic models without
restrictions by subclassing them. E.g. using a third party
model as the root of a polymorphic inheritance tree::

    from thirdparty import ThirdPartyModel

    class MyThirdPartyBaseModel(PolymorphicModel, ThirdPartyModel):
        pass    # or add fields

Or instead integrating the third party model anywhere into an
existing polymorphic inheritance tree::

    class MyBaseModel(SomePolymorphicModel):
        my_field = models.CharField(max_length=10)

    class MyModelWithThirdParty(MyBaseModel, ThirdPartyModel):
        pass    # or add fields


Non-Polymorphic Queries
-----------------------

If you insert ``.non_polymorphic()`` anywhere into the query chain, then
django_polymorphic will simply leave out the final step of retrieving the
real objects, and the manager/queryset will return objects of the type of
the base class you used for the query, like vanilla Django would
(``ModelA`` in this example). 

>>> qs=ModelA.objects.non_polymorphic().all()
>>> qs
[ <ModelA: id 1, field1 (CharField)>,
  <ModelA: id 2, field1 (CharField)>,
  <ModelA: id 3, field1 (CharField)> ]

There are no other changes in the behaviour of the queryset. For example,
enhancements for ``filter()`` or ``instance_of()`` etc. still work as expected.
If you do the final step yourself, you get the usual polymorphic result:

>>> ModelA.objects.get_real_instances(qs)
[ <ModelA: id 1, field1 (CharField)>,
  <ModelB: id 2, field1 (CharField), field2 (CharField)>,
  <ModelC: id 3, field1 (CharField), field2 (CharField), field3 (CharField)> ]


About Queryset Methods
----------------------

*   ``annotate()`` and ``aggregate()`` work just as usual, with the
    addition that the ``ModelX___field`` syntax can be used for the
    keyword arguments (but not for the non-keyword arguments).

*   ``order_by()`` similarly supports the ``ModelX___field`` syntax
    for specifying ordering through a field in a submodel.

*   ``distinct()`` works as expected. It only regards the fields of
    the base class, but this should never make a difference.

*   ``select_related()`` works just as usual, but it can not (yet) be used
    to select relations in inherited models
    (like ``ModelA.objects.select_related('ModelC___fieldxy')`` )

*   ``extra()`` works as expected (it returns polymorphic results) but
    currently has one restriction: The resulting objects are required to have
    a unique primary key within the result set - otherwise an error is thrown
    (this case could be made to work, however it may be mostly unneeded)..
    The keyword-argument "polymorphic" is no longer supported.
    You can get back the old non-polymorphic behaviour
    by using ``ModelA.objects.non_polymorphic().extra(...)``.

*   ``get_real_instances()`` allows you to turn a
    queryset or list  of base model objects efficiently into the real objects.
    For example, you could do ``base_objects_queryset=ModelA.extra(...).non_polymorphic()``
    and then call ``real_objects=base_objects_queryset.get_real_instances()``. Or alternatively
    .``real_objects=ModelA.objects.get_real_instances(base_objects_queryset_or_object_list)``

*   ``values()`` & ``values_list()`` currently do not return polymorphic
    results. This may change in the future however. If you want to use these
    methods now, it's best if you use ``Model.base_objects.values...`` as
    this is guaranteed to not change. 

*   ``defer()`` and ``only()`` are not yet supported (support will be added
    in the future). 


Using enhanced Q-objects in any Places
--------------------------------------

The queryset enhancements (e.g. ``instance_of``) only work as arguments
to the member functions of a polymorphic queryset.  Occasionally it may
be useful to be able to use Q objects with these enhancements in other places.
As Django doesn't understand these enhanced Q objects, you need to
transform them manually into normal Q objects before you can feed them
to a Django queryset or function::

    normal_q_object = ModelA.translate_polymorphic_Q_object( Q(instance_of=Model2B) )

This function cannot be used at model creation time however (in models.py),
as it may need to access the ContentTypes database table.


Nicely Displaying Polymorphic Querysets
---------------------------------------

In order to get the output as seen in all examples here, you need to use the
:class:`~polymorphic.showfields.ShowFieldType` class mixin::

    from polymorphic.showfields import PolymorphicModel, ShowFieldType

    class ModelA(ShowFieldType, PolymorphicModel):
        field1 = models.CharField(max_length=10)

You may also use :class:`~polymorphic.showfields.ShowFieldContent`
or :class:`~polymorphic.showfields.ShowFieldTypeAndContent` to display
additional information when printing querysets (or converting them to text).

When showing field contents, they will be truncated to 20 characters. You can
modify this behaviour by setting a class variable in your model like this::

    class ModelA(ShowFieldType, PolymorphicModel):
        polymorphic_showfield_max_field_width = 20
        ...

Similarly, pre-V1.0 output formatting can be re-estated by using
``polymorphic_showfield_old_format = True``.



.. _restrictions:

Restrictions & Caveats
----------------------

*   Database Performance regarding concrete Model inheritance in general.
    Please see the :ref:`performance`.

*   Queryset methods ``values()``, ``values_list()``, ``select_related()``,
    ``defer()`` and ``only()`` are not yet fully supported (see above).
    ``extra()`` has one restriction: the resulting objects are required to have
    a unique primary key within the result set.

*   Diamond shaped inheritance: There seems to be a general problem 
    with diamond shaped multiple model inheritance with Django models
    (tested with V1.1 - V1.3).
    An example is here: http://code.djangoproject.com/ticket/10808.
    This problem is aggravated when trying to enhance models.Model
    by subclassing it instead of modifying Django core (as we do here
    with PolymorphicModel).

*   The enhanced filter-definitions/Q-objects only work as arguments
    for the methods of the polymorphic querysets. Please see above
    for ``translate_polymorphic_Q_object``.

*   When using the ``dumpdata`` management command on polymorphic tables
    (or any table that has a reference to
    :class:`~django.contrib.contenttypes.models.ContentType`),
    include the ``--natural`` flag in the arguments.



.. old links:
    - http://code.djangoproject.com/wiki/ModelInheritance
    - http://lazypython.blogspot.com/2009/02/second-look-at-inheritance-and.html
    - http://www.djangosnippets.org/snippets/1031/
    - http://www.djangosnippets.org/snippets/1034/
    - http://groups.google.com/group/django-developers/browse_frm/thread/7d40ad373ebfa912/a20fabc661b7035d?lnk=gst&q=model+inheritance+CORBA#a20fabc661b7035d
    - http://groups.google.com/group/django-developers/browse_thread/thread/9bc2aaec0796f4e0/0b92971ffc0aa6f8?lnk=gst&q=inheritance#0b92971ffc0aa6f8
    - http://groups.google.com/group/django-developers/browse_thread/thread/3947c594100c4adb/d8c0af3dacad412d?lnk=gst&q=inheritance#d8c0af3dacad412d
    - http://groups.google.com/group/django-users/browse_thread/thread/52f72cffebb705e/b76c9d8c89a5574f
    - http://peterbraden.co.uk/article/django-inheritance
    - http://www.hopelessgeek.com/2009/11/25/a-hack-for-multi-table-inheritance-in-django
    - http://stackoverflow.com/questions/929029/how-do-i-access-the-child-classes-of-an-object-in-django-without-knowing-the-name/929982#929982
    - http://stackoverflow.com/questions/1581024/django-inheritance-how-to-have-one-method-for-all-subclasses
    - http://groups.google.com/group/django-users/browse_thread/thread/cbdaf2273781ccab/e676a537d735d9ef?lnk=gst&q=polymorphic#e676a537d735d9ef
    - http://groups.google.com/group/django-users/browse_thread/thread/52f72cffebb705e/bc18c18b2e83881e?lnk=gst&q=model+inheritance#bc18c18b2e83881e
    - http://code.djangoproject.com/ticket/10808
    - http://code.djangoproject.com/ticket/7270

