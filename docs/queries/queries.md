# Queries

Making queries is a must when using an ORM and being able to make complex queries is even better
when allowed.

SQLAlchemy is known for its performance when querying a database and it is very fast. The core
being part of **Edgy** also means that edgy performs extremely well when doing it.

When making queries in a [model][model], the ORM uses the [managers][managers] to
perform those same actions.

If you haven't yet seen the [models][model] and [managers][managers] section, now would
be a great time to have a look and get yourself acquainted .

## QuerySet

When making queries within Edgy, this return or an object if you want only one result or a
`queryset` which is the internal representation of the results.

If you are familiar with Django querysets, this is **almost** the same and by almost is because
edgy restricts loosely queryset variable assignments.

Let us get familar with queries.

Let us assume you have the following `User` model defined.

```python
{!> ../docs_src/queries/model.py !}
```

As mentioned before, Edgy returns queysets and simple objects and when queysets are returned
those can be chained together, for example, with `filter()` or `limit()`.

```python
await User.query.filter(is_active=True).filter(first_name__icontains="a").order_by("id")
```

Do we really need two filters here instead of one containing both conditions? No, we do not but
this is for example purposes.

Internally when querying the model and with returning querysets, **Edgy** runs the `all()`.
This can be done manually by you or automatically by the ORM.

Let us refactor the previous queryset and apply the manual `all()`.

```python
await User.query.filter(is_active=True, first_name__icontains="a").order_by("id").all()
```

And that is it. Of course there are more filters and operations that you can do with the ORM and
we will be covering that in this document but in a nutshell, querying the database is this simple.

## Returning querysets

There are many operations you can do with the querysets and then you can also leverage those for
your use cases.

### Exclude

The `exclude()` is used when you want to filter results by excluding instances.

```python
users = await User.query.exclude(is_active=False)
```

### Filter

#### Django-style

These filters are the same **Django-style** lookups.

```python
users = await User.query.filter(is_active=True, email__icontains="gmail")
```

The same special operators are also automatically added on every column.

* **in** - SQL `IN` operator.
* **exact** - Filter instances matching the exact value.
* **iexact** - Filter instances mathing the exact value but case-insensitive.
* **contains** - Filter instances that contains a specific value.
* **icontains** - Filter instances that contains a specific value but case-insensitive.
* **lt** - Filter instances having values `Less Than`.
* **lte** - Filter instances having values `Less Than Equal`.
* **gt** - Filter instances having values `Greater Than`.
* **gte** - Filter instances having values `Greater Than Equal`.

##### Example

```python
users = await User.query.filter(email__icontains="foo")

users = await User.query.filter(id__in=[1, 2, 3])
```

#### SQLAlchemy style

Since Edgy uses SQLAlchemy core, it is also possible to do queries in SQLAlchemy style.
The filter accepts also those.

##### Example

```python
users = await User.query.filter(User.columns.email.contains("foo"))

users = await User.query.filter(User.columns.id.in_([1, 2, 3]))
```

!!! Warning
    The `columns` refers to the columns of the underlying SQLAlchemy table.

All the operations you would normally do in SQLAlchemy syntax, are allowed here.

### Limit

Limiting the number of results. The `LIMIT` in SQL.

```python
users = await User.query.limit(1)

users = await User.query.filter(email__icontains="foo").limit(2)
```

### Offset

Applies the office to the query results.

```python
users = await User.query.offset(1)

users = await User.query.filter(is_active=False).offset(2)
```

Since you can chain the querysets from other querysets, you can aggregate multiple operators in one
go as well.

```python
await User.query.filter(email__icontains="foo").limit(5).order_by("id")
```

### Order by

Classic SQL operation and you need to order results.


**Order by descending id and ascending email**

```python
users = await User.query.order_by("email", "-id")
```

**Order by ascending id and ascending email**

```python
users = await User.query.order_by("email", "id")
```

### Lookup

This is a broader way of searching for a given term. This can be quite an expensive operation so
**be careful when using it**.

```python
users = await User.query.lookup(term="gmail")
```

### Distinct

Applies the SQL `DISTINCT ON` on a table.

```python
users = await User.query.distinct("email")
```

!!! Warning
    Not all the SQL databases support the `DISTINCT ON` fields equally, for example, `mysql` has
    has that limitation whereas `postgres` does not.
    Be careful to know and understand where this should be applied.

### Select related

Returns a QuerySet that will “follow” foreign-key relationships, selecting additional
related-object data when it executes its query.

This is a performance booster which results in a single more complex query but means

later use of foreign-key relationships won’t require database queries.

A simple query:

```python
profiles = await Profile.query.select_related("user")
```

Or adding more operations on the top

```python
profiles = await Profile.query.select_related("user").filter(email__icontains="foo").limit(2)
```

## Returning results

### All

Returns all the instances.

```python
users = await User.query.all()
```

!!! Tip
    The all as mentioned before it automatically executed by **Edgy** if not provided and it
    can also be aggregated with other [queryset operations](#returning-querysets).


### Save

This is a classic operation that is very useful depending on which operations you need to perform.
Used to save an existing object in the database. Slighly different from the [update](#update) and
simpler to read.

```python
await User.query.create(is_active=True, email="foo@bar.com")

user = await User.query.get(email="foo@bar.com")
user.email = "bar@foo.com"

await user.save()
```

Now a more unique, yet possible scenario with a save. Imagine you need to create an exact copy
of an object and store it in the database. These cases are more common than you think but this is
for example purposes only.

```python
await User.query.create(is_active=True, email="foo@bar.com", name="John Doe")

user = await User.query.get(email="foo@bar.com")
# User(id=1)

# Making a quick copy
user.id = None
new_user = await user.save()
# User(id=2)
```

### Create

Used to create model instances.

```python
await User.query.create(is_active=True, email="foo@bar.com")
await User.query.create(is_active=False, email="bar@foo.com")
await User.query.create(is_active=True, email="foo@bar.com", first_name="Foo", last_name="Bar")
```

### Delete

Used to delete an instance.

```python
await User.query.filter(email="foo@bar.com").delete()
```

Or directly in the instance.

```python
user = await User.query.get(email="foo@bar.com")

await user.delete()
```

### Update

You can update model instances by calling this operator.


```python
await User.query.filter(email="foo@bar.com").update(email="bar@foo.com")
```

Or directly in the instance.

```python
user = await User.query.get(email="foo@bar.com")

await user.update(email="bar@foo.com")
```

Or not very common but also possible, update all rows in a table.

```python
user = await User.query.update(email="bar@foo.com")
```

### Get

Obtains a single record from the database.

```python
user = await User.query.get(email="foo@bar.com")
```

You can mix the queryset returns with this operator as well.

```python
user = await User.query.filter(email="foo@bar.com").get()
```

### First

When you need to return the very first result from a queryset.

```python
user = await User.query.first()
```

You can also apply filters when needed.

### Last

When you need to return the very last result from a queryset.

```python
user = await User.query.last()
```

You can also apply filters when needed.

### Exists

Returns a boolean confirming if a specific record exists.

```python
exists = await User.query.filter(email="foo@bar.com").exists()
```

### Count

Returns an integer with the total of records.

```python
total = await User.query.count()
```

### Contains

Returns true if the QuerySet contains the provided object.

```python
user = await User.query.create(email="foo@bar.com")

exists = await User.query.contains(instance=user)
```

### Values

Returns the model results in a dictionary like format.

```python
await User.query.create(name="John" email="foo@bar.com")

# All values
user = User.query.values()
users == [
    {"id": 1, "name": "John", "email": "foo@bar.com"},
]

# Only the name
user = User.query.values("name")
users == [
    {"name": "John"},
]
# Or as a list
# Only the name
user = User.query.values(["name"])
users == [
    {"name": "John"},
]

# Exclude some values
user = User.query.values(exclude=["id"])
users == [
    {"name": "John", "email": "foo@bar.com"},
]
```

The `values()` can also be combined with `filter`, `only`, `exclude` as per usual.

**Parameters**:

* **fields** - Fields of values to return.
* **exclude** - Fields to exclude from the return.
* **exclude_none** - Boolean flag indicating if the fields with `None` should be excluded.

### Values list

Returns the model results in a tuple like format.

```python
await User.query.create(name="John" email="foo@bar.com")

# All values
user = User.query.values_list()
users == [
    (1, "John" "foo@bar.com"),
]

# Only the name
user = User.query.values_list("name")
users == [
    ("John",),
]
# Or as a list
# Only the name
user = User.query.values_list(["name"])
users == [
    ("John",),
]

# Exclude some values
user = User.query.values(exclude=["id"])
users == [
    ("John", "foo@bar.com"),
]

# Flattened
user = User.query.values_list("email", flat=True)
users == [
    "foo@bar.com",
]
```

The `values_list()` can also be combined with `filter`, `only`, `exclude` as per usual.

**Parameters**:

* **fields** - Fields of values to return.
* **exclude** - Fields to exclude from the return.
* **exclude_none** - Boolean flag indicating if the fields with `None` should be excluded.
* **flat** - Boolean flag indicating the results should be flattened.

### Only

Returns the results containing **only** the fields in the query and nothing else.

```python
await User.query.create(name="John" email="foo@bar.com")

user = await User.query.only("name")
```

!!! Warning
    You can only use `only()` or `defer()` but not both combined or a `QuerySetError` is raised.

### Defer

Returns the results containing all the fields **but the ones you want to exclude**.

```python
await User.query.create(name="John" email="foo@bar.com")

user = await User.query.defer("name")
```

!!! Warning
    You can only use `only()` or `defer()` but not both combined or a `QuerySetError` is raised.

### Get or none

When querying a model and do not want to raise a [DoesNotFound](../exceptions.md#doesnotfound) and
instead returns a `None`.

```python
user = await User.query.get_or_none(id=1)
```

## Useful methods

### Get or create

When you need get an existing model instance from the matching query. If exists, returns or creates
a new one in case of not existing.

Returns a tuple of `instance` and boolean `created`.

```python
user, created = await User.query.get_or_create(email="foo@bar.com", defaults={
    "is_active": False, "first_name": "Foo"
})
```

This will query the `User` model with the `email` as the lookup key. If it doesn't exist, then it
will use that value with the `defaults` provided to create a new instance.

!!! Warning
    Since the `get_or_create()` is doing a [get](#get) internally, it can also raise a
    [MultipleObjectsReturned](../exceptions.md#multipleobjectsreturned).


### Update or create

When you need to update an existing model instance from the matching query. If exists, returns or creates
a new one in case of not existing.

Returns a tuple of `instance` and boolean `created`.

```python
user, created = await User.query.update_or_create(email="foo@bar.com", defaults={
    "is_active": False, "first_name": "Foo"
})
```

This will query the `User` model with the `email` as the lookup key. If it doesn't exist, then it
will use that value with the `defaults` provided to create a new instance.

!!! Warning
    Since the `get_or_create()` is doing a [get](#get) internally, it can also raise a
    [MultipleObjectsReturned](../exceptions.md#multipleobjectsreturned).


### Bulk create

When you need to create many instances in one go, or `in bulk`.

```python
await User.query.bulk_create([
    {"email": "foo@bar.com", "first_name": "Foo", "last_name": "Bar", "is_active": True},
    {"email": "bar@foo.com", "first_name": "Bar", "last_name": "Foo", "is_active": True},
])
```

### Bulk update

When you need to update many instances in one go, or `in bulk`.

```python
await User.query.bulk_create([
    {"email": "foo@bar.com", "first_name": "Foo", "last_name": "Bar", "is_active": True},
    {"email": "bar@foo.com", "first_name": "Bar", "last_name": "Foo", "is_active": True},
])

users = await User.query.all()

for user in users:
    user.is_active = False

await User.query.bulk_update(users, fields=['is_active'])
```

## Operators

There are sometimes the need of adding some extra conditions like `AND`, or `OR` or even the `NOT`
into your queries and therefore Edgy provides a simple integration with those.

Edgy provides the [and_](#and), [or_](#or) and [not_](#not) operators directly for you to use, although
this ones come with a slighly different approach.

For all the examples, let us use the model below.

```python
{!> ../docs_src/queries/clauses/model.py !}
```

### SQLAlchemy style

Since Edgy is built on the top of SQL Alchemy core, that also means we can also use directly that
same functionality within our queries.

In other words, uses the [SQLAlchemy style](#sqlalchemy-style).

!!! Warning
    The `or_`, `and_` and `not_` do not work with [related](./related-name.md) operations and only
    directly with the model itself.

This might sound confusing so let us see some examples.

#### AND

As the name suggests, you want to add the `AND` explicitly.

```python
{!> ../docs_src/queries/clauses/and.py !}
```

As mentioned before, applying the [SQLAlchemy style](#sqlalchemy-style) also means you can do this.

```python
{!> ../docs_src/queries/clauses/and_two.py !}
```

And you can do nested `querysets` like multiple [filters](#filter).

```python
{!> ../docs_src/queries/clauses/and_m_filter.py !}
```

#### OR

The same principle as the [and_](#and) but applied to the `OR`.

```python
{!> ../docs_src/queries/clauses/or.py !}
```

As mentioned before, applying the [SQLAlchemy style](#sqlalchemy-style) also means you can do this.

```python
{!> ../docs_src/queries/clauses/or_two.py !}
```

And you can do nested `querysets` like multiple [filters](#filter).

```python
{!> ../docs_src/queries/clauses/or_m_filter.py !}
```

#### NOT

This is simple and direct, this is where you apply the `NOT`.

```python
{!> ../docs_src/queries/clauses/not.py !}
```

As mentioned before, applying the [SQLAlchemy style](#sqlalchemy-style) also means you can do this.

```python
{!> ../docs_src/queries/clauses/not_two.py !}
```

And you can do nested `querysets` like multiple [filters](#filter).

```python
{!> ../docs_src/queries/clauses/not_m_filter.py !}
```

### Edgy Style

This is the most common used scenario where you can use the [related](./related-name.md) for your
queries and all the great functionalities of Edgy while using the operands.

!!! Tip
    The same way you apply the filters for the queries using the [related](./related-name.md), this
    can also be done with the **Edgy style** but the same cannot be said for the
    [SQLAlchemy style](#sqlalchemy-style-1). So if you want to leverage the full power of Edgy,
    it is advised to go Edgy style.

#### AND

The `AND` operand with the syntax is the same as using the [filter](#filter) or any queryset
operatator but for visualisation purposes this is also available in the format of `and_`.

```python
{!> ../docs_src/queries/clauses/style/and_two.py !}
```

With multiple parameters.

```python
{!> ../docs_src/queries/clauses/style/and.py !}
```

And you can do nested `querysets` like multiple [filters](#filter).

```python
{!> ../docs_src/queries/clauses/style/and_m_filter.py !}
```

#### OR

The same principle as the [and_](#and-1) but applied to the `OR`.

```python
{!> ../docs_src/queries/clauses/style/or.py !}
```

With multiple `or_` or nultiple parametes in the same `or_`

```python
{!> ../docs_src/queries/clauses/style/or_two.py !}
```

And you can do nested `querysets` like multiple [filters](#filter).

```python
{!> ../docs_src/queries/clauses/style/or_m_filter.py !}
```

#### NOT

The `not_` as the same principle as the [exclude](#exclude) and like the [and](#and-1), for
representation purposes, Edgy also has that function.

```python
{!> ../docs_src/queries/clauses/style/not.py !}
```

With multiple `not_`.

```python
{!> ../docs_src/queries/clauses/style/not_two.py !}
```

And you can do nested `querysets` like multiple [filters](#filter).

```python
{!> ../docs_src/queries/clauses/style/not_m_filter.py !}
```

Internally, the `not_` is calling the [exclude](#exclude) and applying the operators so this is
more for *cosmetic* purposes than anything else, really.

[model]: ../models.md
[managers]: ../managers.md
