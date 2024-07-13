# Simple yet useful ListView with built-in filtering (in Django)
 
While developing website [energokodros](https://github.com/senyehor/energokodros_website), I needed to create many uniform list views that show some objects from the same model and need filtering by text fields. So, it was obvious this logic needs to be extracted. The best way I found for achieving this is creating a mixin class. Doing so allowed me to make views almost the same as regular ListViews, with minimal additions in view declaration code, explicit and simple. Even though the article is about specific implementation, there are still some cool tips and tricks.

The main logic behind the mixin is to filter objects that are given to a template, which should be usually done in `get_queryset` method. So, I started with creating a mixin scratch that overrides the needed method. So, we the class needs to store filtering fields, and the perform the functionallity to actually filter them. Note that I created a separate __filter class atribute to lessen coupling, not calling some class direcly.
```python
class QuerySetFieldsIcontainsFilterMixin:
    filter_fields: tuple[str, ...] = None
    __filter = None  # we will need to implement filtering logic later

    def get_queryset(self) -> QuerySet:
        if search_value := self.__get_search_value():
            return self.__filter_queryset_for_value(search_value)
        return self.queryset

    def __filter_queryset_for_value(self, value: str) -> QuerySet:
        return self.__filter(
            self.queryset,
            self.filter_fields,
        ).filter(value)

    def __get_search_value(self) -> str:
        return None  # we will need to get value to filter somehow
```
Lets start with the logic part, creating a class that will do the filtration itself. As we see from this part
```python
return self.__filter(
            self.queryset,
            self.filter_fields,
        ).filter(value)
```
it will need to have some queryset, fields to filter againts and the value to filter. There are three parts worth paying attention. The first one is that we convert input fields to a list of `field__icontains`, they way char data could be filtered in Django. The second is creating a Q object, using dict unpacking. It could be only done this way, as Q requires passing arguments by name, so we can not pass
`Q(field=value)`, because it would be filtering againts `user.field`, for example, not `user.name__icontains`, in case provided field is `'name__iconains'`, for example. Make sure you got this, as this is a very useful thing overall. And the last is that we chain multiple `Q`s in `for` cycle. It is effectively `Q() | Q(some_field_name__icontains=value) | Q(some_other_field_name__icontains=value) ... Q(the_last_field_name__icontains=value)`
```python
from django.db.models import Q


class QuerySetFieldsIcontainsFilter:
    def __init__(self, qs: QuerySet, fields_to_filter: Iterable[str]):
        self.__qs = qs
        self.__fields_to_filter = [f'{field}__icontains' for field in fields_to_filter]

    def filter(self, value: str) -> QuerySet:
        q_filters = Q()
        for field in self.__fields_to_filter:
            q_filters = q_filters | Q(**{field: value})
        return self.__qs.filter(q_filters)
```
Before we proceed, let's create typehint for our mixin's `self`, which will help us with auto-completion, allowing access to both ListView and out mixin's methods and attributes. Note that mixin name is inside quotas, as we have to create TypeAlias before the class is created in order to use it inside class. Python is smart enough to recognize that we are referencing a class below. The name starts with underscore to prevent in from being imported, as it should not to be. Double underscore will not work, as the type alias will not be accessible from mixin method declarations. 
```python
_ListViewWithMixinType: TypeAlias = Union[ListView, 'QuerySetFieldsIcontainsFilterMixin']
```
Here is updated class with filter present, filtration logic implemented and simple `search_value` retrieval from url `GET` parameters.
```python
_ListViewWithMixinType: TypeAlias = Union[ListView, 'QuerySetFieldsIcontainsFilterMixin']


class QuerySetFieldsIcontainsFilterMixin:
    filter_fields: tuple[str, ...] = None
    __filter = QuerySetFieldsIcontainsFilter

    def get_queryset(self: _ListViewWithMixinType) -> QuerySet:
        if search_value := self.__get_search_value():
            return self.__filter_queryset_for_value(search_value)
        return self.queryset

    def __filter_queryset_for_value(self: _ListViewWithMixinType, value: str) -> QuerySet:
        return self.__filter(
            self.queryset,
            self.filter_fields,
        ).filter(value)

    def __get_search_value(self: _ListViewWithMixinType) -> str:
        return self.request.GET.get('search_value', None)        
```
That is it with basic mixin functionality, now it can be used simply like this. The tricky detail is that we have to put our mixin before `ListView`, so it should be extracted to a separate class, so a view inherits from a single class. 
```python
class UserListView(QuerySetFieldsIcontainsFilterMixin, ListView):
    queryset = User.objects.all()
    filter_fields = ('full_name', 'email')
    template_name = 'users/users_list.html'
```
Some more functionallity was needed from the class I used, so let's go through that as well. Firs of all, I also needed to add ordering by fields, so it was implemented the following way. `fields_order_by_before_pk` attribute was added, and additional ordering logic is present now.  
```python
class QuerySetFieldsIcontainsFilterPkOrderedMixin:
    filter_fields: tuple[str, ...] = None
    fields_order_by_before_pk: tuple[str, ...] = tuple()
    __filter = QuerySetFieldsIcontainsFilter

    def get_queryset(self: _ListViewWithMixinType) -> QuerySet:
        if search_value := self.__get_search_value():
            qs = self.__filter_queryset_for_value(search_value)
        else:
            qs = self.queryset
        return qs.order_by(*self.fields_order_by_before_pk, '-pk')
```
And the last touch is to add a class that is "pre-mixed", adding an underscore to the mixin class name to discourage it's direct usage.
```python
class _QuerySetFieldsIcontainsFilterPkOrderedMixin:
    filter_fields: tuple[str, ...] = None
    fields_order_by_before_pk: tuple[str, ...] = tuple()
    __filter = QuerySetFieldsIcontainsFilter

    def get_queryset(self: _ListViewWithMixinType) -> QuerySet:
        if search_value := self.__get_search_value():
            qs = self.__filter_queryset_for_value(search_value)
        else:
            qs = self.queryset
        return qs.order_by(*self.fields_order_by_before_pk, '-pk')

    def __filter_queryset_for_value(self: _ListViewWithMixinType, value: str) -> QuerySet:
        return self.__filter(
            self.queryset,
            self.filter_fields,
        ).filter(value)

    def __get_search_value(self: ListView) -> str:
        return self.request.GET.get('search_value', None)


class ListViewWithFiltering(_QuerySetFieldsIcontainsFilterPkOrderedMixin, ListView):
    pass
```
This allows easier use, without the need to keep in mind that the mixin should be placed first. Now it can be used like that.
```python
class UserListView(ListViewWithFiltering):
    queryset = User.objects.all()
    filter_fields = ('full_name', 'email')
    template_name = 'users/users_list.html'
```
I also would like to mention that there is an althernative way to produce `q_filters`. It requires understanding how functools.reduce work, makind the process much less explicit, so I would not recomment doing it, just a mention how it could be done as well.  
```python
class QuerySetFieldsIcontainsFilter:
    def __init__(self, qs: QuerySet, fields_to_filter: Iterable[str]):
        self._qs = qs
        self._fields_to_filter = [f'{field}__icontains' for field in fields_to_filter]

    def filter(self, value: str) -> QuerySet:
        q_filters = functools.reduce(
            lambda q, field: q | Q(**{field: value}),
            self._fields_to_filter,
            Q()
        )
        return self._qs.filter(q_filters)
```
That is it for this article. I hope you learned something new from than, and thank you for reading. I am open to feedback, so leave a comment or write directly to senyehor@gmail.com.