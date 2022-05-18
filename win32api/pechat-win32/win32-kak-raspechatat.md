# Win32 как распечатать?

{% hint style="info" %}
**Оригинальное название**: Win32 How Do I...? - Print

**Ссылка**: [http://timgolden.me.uk/python/win32\_how\_do\_i.html](http://timgolden.me.uk/python/win32\_how\_do\_i.html)

**Автор**: Tim Golden

**Дата**: ???
{% endhint %}

### Введение

Требование: распечатать

Это, вероятно, самый широкий вопрос, который мне придется здесь затронуть, и тот, в котором существует наибольшее несоответствие между количеством и сложностью решений и простотой требования. Ответ таков: все зависит от того, что вы пытаетесь напечатать, какие инструменты у вас есть и какой уровень контроля вам нужен.

* Если у вас просто есть «документ» (читай: файл известного типа, связанный с одним приложением), который вы хотите распечатать, и вы не слишком привередливы в управлении, вы можете [использовать подход ShellExecute](win32-kak-raspechatat.md#standartnyi-dokument-ispolzuite-shellexecute). Это работает (при условии, что у вас установлены соответствующие приложения) для документов Microsoft Office, PDF-файлов, текстовых файлов и практически любых основных приложений. Попробуйте и посмотрите.
* Следующий наиболее распространенный случай — когда у вас есть что-то, например текстовый файл или необработанный PCL, который, как вы знаете, можно отправить непосредственно на принтер. В этом случае вы можете напрямую [использовать функции win32print](win32-kak-raspechatat.md#neobrabotannye-dannye-dlya-pechati-ispolzuite-win32print-napryamuyu).
* Если у вас есть изображение для печати, вы можете объединить возможности [Python Imaging Library](https://pypi.org/project/Pillow/) с модулем win32ui, чтобы выполнить [черновую, но полезную печать](win32-kak-raspechatat.md#odno-izobrazhenie-ispolzuite-pil-i-win32ui) на любом принтере.
* Если у вас есть достаточный объем текста для печати, лучше всего использовать [Reportlab PDF Toolkit](https://www.reportlab.com/opensource/) и его систему документов Platypus для [создания читаемых PDF-файлов](win32-kak-raspechatat.md#mnogo-teksta-sozdaite-pdf) из любого объема текста, а затем использовать технику [ShellExecute](win32-kak-raspechatat.md#standartnyi-dokument-ispolzuite-shellexecute) для его печати.

### Стандартный документ: используйте ShellExecute

Воспользуйтесь тем фактом, что в Win32 типы файлов (по сути, расширения) могут быть связаны с приложениями с помощью командных слов. Как правило, одно и то же приложение будет обрабатывать все команды (обычно это команды **Open** и **Print**), но это не обязательно. Это означает, что вы можете сказать ОС взять ваш файл и вызвать все необходимое для его печати.

#### Плюсы:

* Заботится о стандартных типах файлов
* Не нужно возиться со списками принтеров

#### Минусы:

* Не дает вам контроля
* Работает только для четко определенных пар документ-приложение.
* ~~Печатает только на принтер по умолчанию~~

ОБНОВЛЕНИЕ: Спасибо Крису Курви за то, что он указал, что вы можете указать принтер, включив его с помощью переключателя **d**: в разделе параметров. Не знаю, работает ли это для каждого типа файлов.

```python
import tempfile
import win32api
import win32print

filename = tempfile.mktemp (".txt")
open (filename, "w").write ("This is a test")
win32api.ShellExecute (
  0,
  "print",
  filename,
  #
  # Если это None, то будет использован принтер по умолчанию всегда.
  #
  '/d:"%s"' % win32print.GetDefaultPrinter(),
  ".",
  0
)
```

ОБНОВЛЕНИЕ 2: Мэт Бейкер и Майкл «micolous» указывают, что существует недостаточно документированная [команда printto](https://docs.microsoft.com/en-us/previous-versions/bb776820\(v=vs.85\)?redirectedfrom=MSDN#class), которая принимает имя принтера в качестве параметра, заключенного в кавычки, если оно содержит пробелы. У меня это не работает, но они оба сообщают об успехе, по крайней мере, для некоторых типов файлов.

```python
import tempfile
import win32api
import win32print

filename = tempfile.mktemp (".txt")
open (filename, "w").write ("This is a test")
win32api.ShellExecute (
  0,
  "printto",
  filename,
  '"%s"' % win32print.GetDefaultPrinter(),
  ".",
  0
)
```

### Необработанные данные для печати: используйте win32print напрямую

Модуль **win32print** предлагает (почти) все примитивы печати, которые вам понадобятся, чтобы взять некоторые данные и отправить их на принтер, который уже определен в вашей системе. Данные должны быть в форме, которую принтер с радостью проглотит, обычно что-то вроде текста или необработанного PCL.

#### Плюсы:

* Быстро и просто
* Вы можете решить, какой принтер использовать

Минусы:

* Данные должны быть готовы к печати

```python
import os, sys
import win32print
printer_name = win32print.GetDefaultPrinter()
#
# raw_data также может быть необработанным PCL/PS,
# считанным из какой-либо операции печати в файл.
#
if sys.version_info >= (3,):
  raw_data = bytes("This is a test", "utf-8")
else:
  raw_data = "This is a test"

hPrinter = win32print.OpenPrinter(printer_name)
try:
  hJob = win32print.StartDocPrinter(
    hPrinter,
    1,
    ("test of raw data", None, "RAW")
  )
  try:
    win32print.StartPagePrinter(hPrinter)
    win32print.WritePrinter(hPrinter, raw_data)
    win32print.EndPagePrinter(hPrinter)
  finally:
    win32print.EndDocPrinter(hPrinter)
finally:
  win32print.ClosePrinter(hPrinter)
```

### Одно изображение: используйте PIL и win32ui

Без каких-либо дополнительных инструментов печать изображения на компьютере с Windows почти безумно сложна, включая как минимум три контекста устройств, связанных друг с другом на разных уровнях, и изрядное количество проб и ошибок. К счастью, существует аппаратно-независимое растровое изображение (DIB), которое позволяет разрубить гордиев узел — или, по крайней мере, частично. К счастью, Python Imaging Library поддерживает зверя. Следующий код быстро берет файл изображения и принтер и печатает изображение максимально большого размера на странице без потери соотношения сторон, что требуется в большинстве случаев.

#### Плюсы:

* Оно работает
* Вы можете решить, какой принтер использовать
* (Спасибо PIL) Вы можете выбрать множество форматов изображений

#### Минусы:

* Если вы не знакомы с контекстами устройств Windows, это не самый понятный метод.

```python
import win32print
import win32ui
from PIL import Image, ImageWin
#
# Константы для GetDeviceCaps
#
#
# HORZRES / VERTRES = область печати
#
HORZRES = 8
VERTRES = 10
#
# LOGPIXELS = точек на дюйм
#
LOGPIXELSX = 88
LOGPIXELSY = 90
#
# PHYSICALWIDTH/HEIGHT = Общая площадь
#
PHYSICALWIDTH = 110
PHYSICALHEIGHT = 111
#
# PHYSICALOFFSETX/Y = левое/верхнее поле
#
PHYSICALOFFSETX = 112
PHYSICALOFFSETY = 113

printer_name = win32print.GetDefaultPrinter()
file_name = "test.jpg"

#
# Вы можете записать независимое от устройства растровое изображение
# непосредственно в контекст устройства Windows;
# поэтому нам нужно (для простоты) использовать Python Imaging Library
# для управления изображением.
#
# Создайте контекст устройства из именованного принтера
# и оцените размер бумаги для печати.
#
hDC = win32ui.CreateDC()
hDC.CreatePrinterDC(printer_name)
printable_area = hDC.GetDeviceCaps(HORZRES), hDC.GetDeviceCaps(VERTRES)
printer_size = hDC.GetDeviceCaps(PHYSICALWIDTH), hDC.GetDeviceCaps(PHYSICALHEIGHT)
printer_margins = hDC.GetDeviceCaps(PHYSICALOFFSETX), hDC.GetDeviceCaps(PHYSICALOFFSETY)

#
# Откройте изображение, поверните его, если оно в ширину больше,
# чем в высоту, и решите, на сколько умножить каждый пиксель,
# чтобы получить его как можно больше на странице без искажений.
#
bmp = Image.open (file_name)
if bmp.size[0] > bmp.size[1]:
  bmp = bmp.rotate (90)

ratios = [
  1.0 * printable_area[0] / bmp.size[0],
  1.0 * printable_area[1] / bmp.size[1]
]
scale = min(ratios)

#
# Запустите задание на печать и нарисуйте растровое изображение
# на принтере в масштабированном размере.
#
hDC.StartDoc(file_name)
hDC.StartPage()

dib = ImageWin.Dib(bmp)
scaled_width, scaled_height = [int (scale * i) for i in bmp.size]
x1 = int ((printer_size[0] - scaled_width) / 2)
y1 = int ((printer_size[1] - scaled_height) / 2)
x2 = x1 + scaled_width
y2 = y1 + scaled_height
dib.draw(hDC.GetHandleOutput(), (x1, y1, x2, y2))

hDC.EndPage ()
hDC.EndDoc ()
hDC.DeleteDC ()
```

Учитывая технику (создание контекста устройства), вы можете использовать на нем любые стандартные функции Windows, такие как DrawText, BitBlt и т. д. Например (пример из документации pywin32):

```python
import win32ui
import win32print
import win32con

INCH = 1440

hDC = win32ui.CreateDC ()
hDC.CreatePrinterDC(win32print.GetDefaultPrinter())
hDC.StartDoc("Test doc")
hDC.StartPage()
hDC.SetMapMode(win32con.MM_TWIPS)
hDC.DrawText("TEST", (0, INCH * -1, INCH * 8, INCH * -2), win32con.DT_CENTER)
hDC.EndPage()
hDC.EndDoc()
```

### Много текста: создайте PDF

Вы можете просто отправить текст прямо на принтер, но вы зависите от шрифтов, полей и того, что у вас есть, которые определил принтер. Вместо того, чтобы создавать необработанные коды PCL, вы можете создавать PDF-файлы, а Acrobat позаботится о печати. Инструментарий Reportlab делает это в высшей степени хорошо, и особенно его структура документов Platypus, которая дает вам возможность создавать документы произвольной сложности. Приведенный ниже пример едва затрагивает поверхность инструментария, но показывает, что вам не нужны две страницы установочного кода для создания удобного PDF-файла. Как только это сгенерировано, вы можете использовать описанную выше технику [ShellExecute](win32-kak-raspechatat.md#standartnyi-dokument-ispolzuite-shellexecute) для печати.

```python
from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer
from reportlab.lib.styles import getSampleStyleSheet
from reportlab.lib.units import inch

import cgi
import tempfile
import win32api

source_file_name = "c:/temp/temp.txt"
pdf_file_name = tempfile.mktemp (".pdf")

styles = getSampleStyleSheet()
h1 = styles["h1"]
normal = styles["Normal"]

doc = SimpleDocTemplate(pdf_file_name)
#
# reportlab ожидает увидеть XML-совместимые данные;
# нужно избегать амперсандов &c.
#
text = cgi.escape(open(source_file_name).read()).splitlines()

#
# Возьмите первую строку документа в качестве заголовка;
# остальные обрабатываются как основной текст.
#
story = [Paragraph(text[0], h1)]
for line in text[1:]:
  story.append(Paragraph(line, normal))
  story.append(Spacer(1, 0.2 * inch))

doc.build (story)
win32api.ShellExecute (0, "print", pdf_file_name, None, ".", 0)
```
