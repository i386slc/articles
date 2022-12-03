# Детальное понимание модельных Django formets и их расширенное использование

{% hint style="info" %}
Ссылка на оригинальную статью: [Understanding Django model formsets in detail and their advanced usage](https://micropyramid.com/blog/understanding-djangos-model-formsets-in-detail-and-their-advanced-usage/)

Опубликовано: 5 мая 2012

Автор: [MicroPyramid](https://micropyramid.com/)
{% endhint %}

Подобно обычным наборам форм, Django также предоставляет набор форм модели, который упрощает работу с моделями Django. Наборы форм моделей Django позволяют редактировать или создавать несколько экземпляров модели в одной форме. Наборы форм модели создаются фабричным методом. Фабричный метод по умолчанию — **modelformset\_factory()**. Он обертывает фабрику наборов форм для моделирования форм. Мы также можем создать **inlineformset\_factory()** для редактирования связанных объектов. **inlineformset\_factory** оборачивает **modelformset\_factory**, чтобы ограничить набор запросов и установить исходные данные для связанных объектов экземпляра.

## Шаг 1: Создание модели в models.py

```python
class User(models.Model):
    first_name = models.CharField(max_length=150)
    last_name = models.CharField(max_length=150)
    user_group = models.ForeignKey(Group)
    birth_date = models.DateField(blank=True, null=True)
```

## Шаг 2: в forms.py

```python
from django.forms.models import modelformset_factory
from myapp.models import User

UserFormSet = modelformset_factory(User, exclude=())
```

Это создаст набор форм, способный работать с данными, связанными с моделью пользователя **User**. Мы также можем передать данные набора запросов в набор форм модели, чтобы он мог вносить изменения только в данный набор запросов.

```python
formset = UserFormSet(queryset=User.objects.filter(first_name__startswith='M'))
```

Мы можем создать дополнительную форму в шаблоне, передав аргумент **extra** методу **modelformset\_factory**, мы можем использовать это следующим образом.

```python
UserFormSet = modelformset_factory(User, exclude=(), extra=1)
```

Мы можем настроить форму, которая будет отображаться в шаблоне, передав новую настроенную форму в **modelformset\_factory**. Например: в нашем текущем примере, если вы хотите, чтобы **birthday\_date** был виджетом выбора даты, мы можем добиться этого с помощью следующего изменения в нашем **form.py**.

```python
class UserForm(forms.ModelForm):

    birth_date = forms.DateField(
        widget=DateTimePicker(options={"format": "YYYY-MM-DD", "pickSeconds": False})
    )
    class Meta:
        model = User
        exclude = ()

UserFormSet = modelformset_factory(User, form=UserForm)
```

В общем, наборы моделей Django выполняют проверку, когда заполнен хотя бы один из данных, в большинстве случаев нам понадобится сценарий, в котором нам требуется добавить хотя бы один объект данных, или другой сценарий, в котором нам потребуется передать некоторые начальные данные в форму, мы можем добиться такого рода случаев, переопределив **basemodelformset** следующим образом.

> **business\_profile\_id** - идентификатор группы, с которой мы хотим работать. То есть мы хотим сразу передать в форму только те данные, с которыми нужно работать.
>
> **\_construct\_form()** - в FormSet перед конструированием формы (_скорее всего_) сразу происходит заполнение поля необходимыми данными.
>
> **empty\_permitted** - `formset[0].empty_permitted` означает, что если `formset[0].has_changed() == False`, дальнейшая обработка/проверка **не выполняется**. Чтобы предотвратить это - `empty_permitted = False`.

В **forms.py**:

```python
class UserForm(forms.ModelForm):

    birth_date = forms.DateField(
        widget=DateTimePicker(options={"format": "YYYY-MM-DD", "pickSeconds": False})
    )
    class Meta:
        model = User
        exclude = ()

    def __init__(self, *args, **kwargs):
        self.businessprofile_id = kwargs.pop('businessprofile_id')
        super(UserForm, self).__init__(*args, **kwargs)

        self.fields['user_group'].queryset = Group.objects.filter(
            business_profile_id = self.businessprofile_id
        )

BaseUserFormSet = modelformset_factory(User, form=UserForm, extra=1, can_delete=True)

class UserFormSet(BaseUserFormSet):

    def __init__(self, *args, **kwargs):
        # создайте атрибут пользователя и удалите его из kwargs,
        # чтобы он не путался с другими kwargs набора форм
        self.businessprofile_id = kwargs.pop('businessprofile_id')
        super(UserFormSet, self).__init__(*args, **kwargs)
        for form in self.forms:
            form.empty_permitted = False

    def _construct_form(self, *args, **kwargs):
        # ввести пользователя в каждую форму в наборе форм
        kwargs['businessprofile_id'] = self.businessprofile_id
        return super(UserFormSet, self)._construct_form(*args, **kwargs)
```

## Шаг 3: в views.py

```python
from myapp.forms import UserFormSet
from django.shortcuts import render_to_response

def manage_users(request):

    if request.method == 'POST':
        formset = UserFormSet(businessprofile_id=businessprofileid, data=request.POST)
        if formset.is_valid():
            formset.save()
            # сделать что-нибудь
    else:
        formset = UserFormSet(businessprofile_id=businessprofileid)
    return render_to_response("manage_users.html", {"formset": formset})
```

## Шаг 4: в шаблоне

Самый простой способ отобразить ваш набор форм заключается в следующем.

```django
<form method="post" action="">
      {{ formset }}
</form>
```
