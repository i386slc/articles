# "Добавление" объектов Q в Django

{% hint style="info" %}
Ссылка на оригинальную статью: ["Adding" Q objects in Django](http://bradmontgomery.blogspot.com/2009/06/adding-q-objects-in-django.html)

Опубликовано: 26 июня 2009

Автор: неизвестно
{% endhint %}

У меня есть приложение Django со следующей моделью Model:

```python
class Story(models.Model):
    title = models.CharField(max_length=255)
    content = models.TextField()
```

### Проблема

Я хотел создать простую функцию поиска, которая объединяла бы все условия поиска по схеме **OR**. По сути, я хотел, чтобы SQL выглядел следующим образом:

```sql
SELECT * from myapp_stories where 
    title LIKE '%term1%' OR content LIKE '%term1%' OR 
    title LIKE '%term2%' OR content LIKE '%term2%'; 
```

### Решение

Вы можете добавлять **add** объекты Q django вместе! Эта функция в настоящее время не обсуждается в документации, но я просмотрел исходный код и обнаружил, что объект Q на самом деле является просто узлом в дереве! В частности, Q является подклассом `django.utils.tree.Node` (проверьте, это круто!) У узла есть атрибут, называемый **connector**. Объекты Q имеют два возможных коннектора: **AND** и **OR**. Но как нам связать объекты Q? Что ж, у узла есть удобный метод `add(node, conn_type)`, параметры которого включают другой узел и тип соединения.

Как упоминалось ранее, возможные типы соединения для объектов Q — **AND** и **OR**, поэтому объекты Q можно сложить вместе, выполнив что-то вроде этого:

```python
# AND Q объекты
q_object = Q()
q_object.add(Q(), Q.AND)

# OR Q объекты
q_object = Q()
q_object.add(Q(), Q.OR)
```

Итак, решение моего представления поиска выглядит следующим образом:

```python
from django.db.models import Q
from models import Story

def search(request):
    ''' 
    Общий поиск: GET должен содержать следующее: 
    terms - ключевые слова для поиска, разделенные пробелами
    '''
    terms = request.GET.get('terms', None)
    term_list = terms.split(' ')

    stories = Story.objects.all()

    q = Q(content__icontains=term_list[0]) | Q(title__icontains=term_list[0])
    for term in term_list[1:]:
        q.add((Q(content__icontains=term) | Q(title__icontains=term)), q.connector)

    stories = stories.filter(q)

    return render_to_response('myapp/search.html', locals(), \
            context_instance=RequestContext(request))
```

Излишне говорить, что объекты Q довольно мощные!
