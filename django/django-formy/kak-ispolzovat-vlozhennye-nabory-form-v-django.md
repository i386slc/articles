# Как использовать вложенные наборы форм в django

{% hint style="info" %}
Ссылка на оригинальную статью: [How to use nested formsets in django](https://micropyramid.com/blog/how-to-use-nested-formsets-in-django/)

Опубликовано: 8 мая 2017

Автор: [MicroPyramid](https://micropyramid.com/)
{% endhint %}

Наборы форм Django управляют сложностью нескольких копий формы в представлении. Используя наборы форм, вы можете узнать, сколько форм было изначально, какие из них были изменены, а какие следует удалить.

Подобно формам и формам моделей, Django предлагает наборы форм моделей, которые упрощают задачу создания набора форм для формы, которая обрабатывает несколько экземпляров модели.

Django также предоставляет встроенные наборы форм, которые можно использовать для обработки набора объектов, принадлежащих общему внешнему ключу.

В приведенных ниже примерах моделей мы можем написать встроенный набор форм для обработки всех дочерних элементов для родителя или всех адресов дочернего элемента.

```python
# models.py

class Parent(models.Model):
    name = models.CharField(max_length=255)

class Child(models.Model):
    parent = models.ForeignKey(Parent)
    name = models.CharField(max_length=255)

class Address(models.Model):
    child = models.ForeignKey(Child)
    country = models.CharField(max_length=255)
    state = models.CharField(max_length=255)
    address = models.CharField(max_length=255)
```

```python
# forms.py

from django.forms.models import inlineformset_factory

ChildrenFormset = inlineformset_factory(models.Parent, models.Child, extra=1)
AddressFormset = inlineformset_factory(models.Child, models.Address, extra=1)
```

Используя вышеуказанные наборы форм, вы можете обрабатывать все дочерние элементы для родителя на одной странице и можете обрабатывать все адреса дочернего элемента на другой странице. Но если вы хотите разрешить пользователям добавлять/редактировать все дочерние формы вместе с адресами, все на одной странице, то в этом случае у вас должен быть полный набор адресных форм для каждой дочерней формы в дочернем наборе форм.

Здесь наступает момент использования вложенных наборов форм. Вложенный набор форм — это обычный встроенный набор форм. Следующие шаги помогут вам справиться с вложенными наборами форм.

## Шаг 1: Создайте базовый встроенный набор форм

```python
# forms.py

from django.forms.models import BaseInlineFormSet

class BaseChildrenFormset(BaseInlineFormSet):
    pass

ChildrenFormset = inlineformset_factory(models.Parent,
                                        models.Child,
                                        formset=BaseChildrenFormset,
                                        extra=1)
```

## Шаг 2. Прикрепите вложенный набор форм для каждой формы

Прикрепите вложенный набор форм для каждой формы, как показано ниже. Суперкласс **BaseInlineFormSet** определяет метод **add\_fields**, который отвечает за добавление полей для каждой формы в наборе форм. Итак, здесь мы можем написать логику, чтобы связать вложенный набор форм.

```python
# forms.py
class BaseChildrenFormset(BaseInlineFormSet):

    def add_fields(self, form, index):
        super(BaseChildrenFormset, self).add_fields(form, index)

        # сохранить набор форм в свойстве nested
        form.nested = AddressFormset(
                        instance=form.instance,
                        data=form.data if form.is_bound else None,
                        files=form.files if form.is_bound else None,
                        prefix='address-%s-%s' % (
                            form.prefix,
                            AddressFormset.get_default_prefix()),
                        extra=1)
```

{% hint style="info" %}
Здесь мы создали новое свойство под названием **form.nested**, которое содержит набор вложенных форм (**AddressFormset**).
{% endhint %}

## Шаг 3. Обработка набора форм и вложенных наборов форм в представлениях

```python
# views.py

def manage_children(request, parent_id):
    """Редактировать детей и их адреса для одного родителя."""

    parent = get_object_or_404(models.Parent, id=parent_id)

    if request.method == 'POST':
        formset = forms.ChildrenFormset(request.POST, instance=parent)
        if formset.is_valid():
            formset.save()
            return redirect('parent_view', parent_id=parent.id)
    else:
        formset = forms.ChildrenFormset(instance=parent)

    return render(request, 'manage_children.html', {
                  'parent':parent,
                  'children_formset':formset})
```

## Шаг 4: Отобразите вложенный набор форм в шаблоне

```django
{# manage_children.html (Просто часть отображения формы) #}

{{ children_formset.management_form }}
{{ children_formset.non_form_errors }}

{% raw %}
{% for child_form in children_formset.forms %}
    {{ child_form }}

    {% if child_form.nested %}
        {{ child_form.nested.management_form }}
        {{ child_form.nested.non_form_errors }}

        {% for nested_form in child_form.nested.forms %}
            {{ nested_form }}
        {% endfor %}

    {% endif %}

{% endfor %}
{% endraw %}
```

## Здесь есть несколько случаев, которые нужно обработать

1. Валидация. При проверке формы в наборе форм нам также необходимо проверить ее подформы, которые находятся во вложенном наборе форм.
2. Сохранение данных. При сохранении формы также необходимо сохранить дополнения/изменения форм во вложенном наборе форм.

Когда страница отправлена, мы вызываем `formset.is_valid()` для проверки форм. Мы переопределяем **is\_valid** в нашем наборе форм, чтобы добавить проверку и для вложенных наборов форм.

```python
# forms.py

class BaseChildrenFormset(BaseInlineFormSet):
    ...

    def is_valid(self):
        result = super(BaseChildrenFormset, self).is_valid()

        if self.is_bound:
            for form in self.forms:
                if hasattr(form, 'nested'):
                    result = result and form.nested.is_valid()

        return result
```

На этом проверка форм и вложенных наборов форм завершается. Теперь нам нужно обработать сохранение. Поэтому для этого нам нужно переопределить метод **save** для сохранения родительского набора форм и всех вложенных наборов форм.

```python
# forms.py

class BaseChildrenFormset(BaseInlineFormSet):
    ...

    def save(self, commit=True):

        result = super(BaseChildrenFormset, self).save(commit=commit)

        for form in self.forms:
            if hasattr(form, 'nested'):
                if not self._should_delete_form(form):
                    form.nested.save(commit=commit)

        return result
```

Метод **save** отвечает за сохранение форм в наборе форм, а также всех форм во вложенном наборе форм для каждой формы.

Чтобы узнать больше о нашем пакете с открытым исходным кодом Django CRM (Customer Relationship Management). [Посмотреть код](https://github.com/MicroPyramid/Django-CRM).
