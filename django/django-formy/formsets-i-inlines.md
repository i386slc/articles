# Formsets и Inlines

{% hint style="info" %}
Ссылка на оригинальную статью: [Formsets and Inlines](http://docs.viewflow.io/forms\_formsets.html)

Опубликовано: неизвестно

Автор: [http://viewflow.io/](http://viewflow.io/)
{% endhint %}

Поддержка наборов форм и встроенных строк основана на идее проекта [django-superform](https://github.com/gregmuellegger/django-superform). Наборы форм и встроенные строки включены и _**работают как обычные поля формы**_ django.

Такой подход избавляет ваш код представления от деталей, специфичных для формы, и позволяет использовать ту же технику шаблонов из формы **django-material**, что и для всех остальных полей.

Из-за бездействия проекта **django-superforms** код был поглощен **django-material** и теперь поддерживается как часть дистрибутива **django-material Pro**.

## Пример

Чтобы использовать поля **Formset** и **Inlines**, вы должны наследовать от **material.forms.Form**:

```python
from django import forms
from django.forms import formset_factory
from material.forms import Form

class AddressForm(forms.Form):
    line_1 = forms.CharField(max_length=250)
    line_2 = forms.CharField(max_length=250)
    state = forms.CharField(max_length=100)
    city = forms.CharField(max_length=100)
    zipcode = forms.CharField(max_length=10)

    layout = Layout(
        'line_1',
        'line_2',
        'state',
        Row('city', 'zipcode'),
    )

AddressFormSet = formset_factory(AddressForm, extra=3, can_delete=True)


class SignupForm(Form):
    username = forms.CharField(max_length=50)
    first_name = forms.CharField(max_length=250)
    last_name = forms.CharField(max_length=250)
    emails = FormSetField(formset_class=EmailFormSet)
    addresses = FormSetField(formset_class=AddressFormSet)

    layout = Layout(
        'username',
        Row('first_name', 'last_name'),
        'emails',
        Stacked(1, 'addresses'),
    )
```

## API

Это оказалось не то, что нужно, поэтому далее не перевожу...
