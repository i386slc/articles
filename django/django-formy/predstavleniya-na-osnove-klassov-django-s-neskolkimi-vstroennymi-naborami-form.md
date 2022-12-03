# Представления на основе классов Django с несколькими встроенными наборами форм

{% hint style="info" %}
Ссылка на оригинальную статью: [Django class-based views with multiple inline formsets](http://kevindias.com/writing/django-class-based-views-multiple-inline-formsets/)

Опубликовано: 8 июля 2013

Автор: Kevin Dias
{% endhint %}

Это просто краткое объяснение того, как использовать наборы форм встроенной модели Django с универсальными представлениями на основе классов. Обработка форм моделей — это область, в которой имеет смысл использовать представления на основе классов, но существует не так много примеров того, как добавить встроенные наборы форм в этот микс. У [Maxime Haineault](https://web.archive.org/web/20170521112730/http://haineault.com/blog/155/) есть сообщение в блоге, которое очень помогло мне, когда я впервые попытался это сделать, но его подход к созданию встроенных наборов форм в методе **get\_context\_data** не работает, если нам нужно выполнить какую-либо проверку, охватывающую несколько форм в наборе форм. Класс представления, который я здесь использую, более точно следует структуре универсального [CreateView](https://web.archive.org/web/20170521112730/https://docs.djangoproject.com/en/1.5/ref/class-based-views/generic-editing/#createview), поэтому добавление миксинов, превращение этого шаблона в миксин или обновление до новых версий Django с меньшей вероятностью вызовет неприятные сюрпризы. Я написал и протестировал это с помощью **Django 1.5**, но он должен работать для **1.3** и выше.

## Формы и модели

Допустим, на нашем сайте есть список рецептов, которые по сути являются просто списками ингредиентов и списками инструкций по приготовлению этих ингредиентов. Итак, в самом базовом виде наши формы и модели могут выглядеть примерно так.

### models.py

```python
# models.py
from django.db import models

class Recipe(models.Model):
    title = models.CharField(max_length=255)
    description = models.TextField()

class Ingredient(models.Model):
    recipe = models.ForeignKey(Recipe)
    description = models.CharField(max_length=255)

class Instruction(models.Model):
    recipe = models.ForeignKey(Recipe)
    number = models.PositiveSmallIntegerField()
    description = models.TextField()
```

### forms.py

```python
# forms.py
from django.forms import ModelForm
from django.forms.models import inlineformset_factory

from .models import Recipe, Ingredient, Instruction

class RecipeForm(ModelForm):
    class Meta:
        model = Recipe

IngredientFormSet = inlineformset_factory(Recipe, Ingredient)
InstructionFormSet = inlineformset_factory(Recipe, Instruction)
```

## Представление views.py

Наше представление создания рецепта очень похоже на общий **CreateView** Django. Единственное, что нам нужно сделать, это переопределить несколько методов, чтобы наши встроенные наборы форм создавались, проверялись и сохранялись вместе с основной формой рецепта. Нам не нужно переопределять метод **get\_context\_data**, потому что он уже обновляет контекст шаблона с помощью любых аргументов ключевого слова, с которыми вы его вызываете.

```python
# views.py
from django.http import HttpResponseRedirect
from django.views.generic import CreateView

from .forms import IngredientFormSet, InstructionFormSet, RecipeForm
from .models import Recipe


class RecipeCreateView(CreateView):
    template_name = 'recipe_add.html'
    model = Recipe
    form_class = RecipeForm
    success_url = 'success/'

    def get(self, request, *args, **kwargs):
        """
        Обрабатывает запросы GET и создает пустые версии формы
        и ее встроенных наборов форм.
        """
        self.object = None
        form_class = self.get_form_class()
        form = self.get_form(form_class)
        ingredient_form = IngredientFormSet()
        instruction_form = InstructionFormSet()
        return self.render_to_response(
            self.get_context_data(form=form,
                                  ingredient_form=ingredient_form,
                                  instruction_form=instruction_form))

    def post(self, request, *args, **kwargs):
        """
        Обрабатывает запросы POST, создавая экземпляр формы
        и его встроенные наборы форм с переданными переменными POST,
        а затем проверяя их на достоверность.
        """
        self.object = None
        form_class = self.get_form_class()
        form = self.get_form(form_class)
        ingredient_form = IngredientFormSet(self.request.POST)
        instruction_form = InstructionFormSet(self.request.POST)
        if (form.is_valid() and ingredient_form.is_valid() and
            instruction_form.is_valid()):
            return self.form_valid(form, ingredient_form, instruction_form)
        else:
            return self.form_invalid(form, ingredient_form, instruction_form)

    def form_valid(self, form, ingredient_form, instruction_form):
        """
        Вызывается, если все формы допустимы. Создает экземпляр рецепта
        вместе со связанными ингредиентами и инструкциями,
        а затем перенаправляет на страницу успеха.
        """
        self.object = form.save()
        ingredient_form.instance = self.object
        ingredient_form.save()
        instruction_form.instance = self.object
        instruction_form.save()
        return HttpResponseRedirect(self.get_success_url())

    def form_invalid(self, form, ingredient_form, instruction_form):
        """
        Вызывается, если форма недействительна. Повторно отображает
        данные контекста с заполненными данными формами и ошибками.
        """
        return self.render_to_response(
            self.get_context_data(form=form,
                                  ingredient_form=ingredient_form,
                                  instruction_form=instruction_form))
```

## Шаблон

Вот основной шаблон для добавления рецепта. Поскольку для шаблона не имеет значения, используем ли мы представление на основе функций или на основе классов, я включаю его сюда в основном для полноты картины. Скрипт `jquery.formset.js` взят из плагина jQuery [django-dynamic-formset](https://web.archive.org/web/20170521112730/http://code.google.com/p/django-dynamic-formset/), который добавляет кнопки для динамического добавления или удаления встроенных наборов форм. Поскольку у нас есть несколько наборов форм на странице, нам нужно дать каждому префикс и сообщить плагину, какие формы принадлежат каждому набору форм, как описано в [документации django-dynamic-formset](https://web.archive.org/web/20170521112730/http://code.google.com/p/django-dynamic-formset/wiki/Usage#Using\_multiple\_Formsets\_on\_the\_same\_page).

```django
<!DOCTYPE html>
<html>
<head>
    <title>Multiformset Demo</title>
    <script src="{{ STATIC_URL }}js/jquery.min.js"></script>
    <script src="{{ STATIC_URL }}js/jquery.formset.js"></script>
    <script type="text/javascript">
        $(function() {
            $(".inline.{{ ingredient_form.prefix }}").formset({
                prefix: "{{ ingredient_form.prefix }}",
            })
            $(".inline.{{ instruction_form.prefix }}").formset({
                prefix: "{{ instruction_form.prefix }}",
            })
        })
    </script>
</head>

<body>
    <div>
        <h1>Add Recipe</h1>
        <form action="." method="post">
            {% raw %}
{% csrf_token %}
            <div>
                {{ form.as_p }}
            </div>
            <fieldset>
                <legend>Recipe Ingredient</legend>
                {{ ingredient_form.management_form }}
                {{ ingredient_form.non_form_errors }}
                {% for form in ingredient_form %}
                    {{ form.id }}
                    <div class="inline {{ ingredient_form.prefix }}">
                        {{ form.description.errors }}
                        {{ form.description.label_tag }}
                        {{ form.description }}
                    </div>
                {% endfor %}
            </fieldset>
            <fieldset>
                <legend>Recipe instruction</legend>
                {{ instruction_form.management_form }}
                {{ instruction_form.non_form_errors }}
                {% for form in instruction_form %}
                    {{ form.id }}
                    <div class="inline {{ instruction_form.prefix }}">
                        {{ form.number.errors }}
                        {{ form.number.label_tag }}
                        {{ form.number }}
                        {{ form.description.errors }}
                        {{ form.description.label_tag }}
                        {{ form.description }}
                    </div>
                {% endfor %}
{% endraw %}
            </fieldset>
            <input type="submit" value="Add recipe" class="submit" />
        </form>
    </div>
</body>
</html>
```
