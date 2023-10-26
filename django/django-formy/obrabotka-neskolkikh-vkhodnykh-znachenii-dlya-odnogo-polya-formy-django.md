# Обработка нескольких входных значений для одного поля формы Django

{% hint style="info" %}
Ссылка на оригинальную статью: [Handling multiple input values for single Django form field](https://coderwall.com/p/kq1d5a/handling-multiple-input-values-for-single-django-form-field)

Опубликовано: 25 июля 2019

Автор: [mdomans](https://coderwall.com/mdomans)
{% endhint %}

Представьте себе такой сценарий: вам нужно написать форму, которая обрабатывает адрес клиента, с полем для номера телефона, включая код города.

```python
from django import forms

class PhoneForm(forms.Form):
    # очень простой
    number = forms.CharField()
    area_code = forms.CharField()
```

Кажется легко? Что, если вам нужно написать HTML с двумя входными данными: одним для номера телефона и одним для кода города, но в базе данных есть только одно поле для номера телефона с кодом города?

Первым ответом было бы обработать это чистым методом, например:

```python
class PhoneForm(forms.ModelForm):
    number = forms.CharField()
    area_code = forms.CharField()

    def clean(self):
        number = self.cleaned_data['number']
        code = self.cleaned_data['area_code']
        self.cleaned_data['full_number'] = code + number
```

Но это: а) противно б) нерастяжимо. Вы не можете просто повторно использовать любой из этих кодов и продолжать действие, ничего не очищая напрямую в методе clean.

Но есть способ получше — введите Django [MultiValueField](https://docs.djangoproject.com/en/1.7/ref/forms/fields/#multivaluefield).

Чтобы использовать его эффективно, нам нужно создать подклассы [MultiValueField](https://docs.djangoproject.com/en/1.7/ref/forms/fields/#multivaluefield) и [MultiWidget](https://docs.djangoproject.com/en/1.5/ref/forms/widgets/#django.forms.MultiWidget). Затем вы устанавливаете подкласс виджета в качестве виджета подкласса поля, чтобы вы могли обрабатывать как манипуляции с данными, так и их отображение.

И последнее, что следует запомнить: вам необходимо реализовать методы сжатия **compress** и распаковки **decompress** для поля и виджета соответственно. Это связано с тем, что поле должно возвращать одно значение и ему присваивается одно значение, независимо от того, сколько входных данных вы фактически отображаете или сколько данных отправляется.

Например:

```python
class PhoneNumberWidget(forms.MultiWidget):
    def __init__(self, *args, **kwargs):
        super(PhoneNumberWidget, self).__init__(*args, **kwargs)
        self.widgets = [
            forms.TextInput(),
            forms.TextInput()
        ]
    def decompress(self, value):
        if value:
            return value.split(' ')
        return [None, None]

class PhoneNumberField(forms.MultiValueField):
    widget = PhoneNumberWidget

    def __init__(self, *args, **kwargs):
        super(PhoneNumberField, self).__init__(*args, **kwargs)
        fields = (
            forms.CharField(),
            forms.CharField()
        )

    def compress(self, data_list):
        return ' '.join(data_list)


class PhoneForm(forms.ModelForm):
    phone_number = PhoneNumberField()

    def __init__(self, *args, **kwargs):
        super(PhoneForm, self).__init__(self, *args, **kwargs)
        self.initial['phone_number'] = ['+1','11111111']
```

Это немного больше кода, но он позволяет красиво обернуть бизнес-логику кодом. Если вам нужно поле для обработки номера телефона с кодом города где-либо еще, вы можете просто импортировать поле вместо использования метода копирования и вставки.
