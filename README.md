## Туториал по использованию
- [Установка](#установка)
- [Инструкция](#инструкция)
    - [Основная концепция](#основная-концепция)
    - [Писатели](#писатели)
    - [Аугментации](#аугментации)
    - [Генераторы](#генераторы)
        - [BackgroundGenerator](#backgroundgenerator)
        - [TextGenerator](#textgenerator)
        - [OverlayGenerator](#overlaygenerator)
    - [Регулирование параметров генерации с помощью constraints](#регулирование-параметров-генерации-с-помощью-constraints)
    - [Зоопарк настроенных генераторов](#зоопарк-настроенных-генераторов)
    - [Как создать свои шрифты или фоны](#как-добавить-шрифты-или-фоны)
        - [Шрифты](#шрифты)
        - [Фоны](#фоны)
        - [Загрузка шрифтов и фонов на s3](#загрузка-шрифтов-и-фонов-на-s3)
        - [Как забрать шрифты и фоны с s3](#как-забрать-шрифты-и-фоны-с-s3)
    - [Пример сборки генератора](#пример-сборки-генератора)



## Установка

Генератор является частью pip-пакета cftcv, чтобы установить cftcv выполните:
```
pip install cftcv==0.1.2rc18 -i http://rpm.ftc.ru/nexus/repository/pypi/simple --trusted-host rpm.ftc.ru
```
`0.1.2rc19` - актуальная стабильная версия на 15.07.2020.

<br>

Чтобы импортировать пакет, используйте:
```python
from cftcv import ocr_generator
```

## Инструкция

### Основная концепция
[↑ up](#туториал-по-использованию)

Существует несколько сущностей:

 - Писатели (наследники BaseWriter) - объекты этих классов "сочиняют" текст по заданным правилам 
 (даты, фио, номера и тд)
 
 - Аугментации (наследники BaseTransforms) - преобразуют изображение (используются генераторами)
 
 - Генераторы (наследники BaseGenerators) - генерируют изображения, используя указанных писателей и аугментации
 
 Каждый базовый класс наследуется от Flexible, навязывающего интерфейс генерации параметров инстанса, 
 их извлечения и установки. 
 
 <img src='./docs/static/images/class_diagram.jpg' alt="class diagram">
 
 Любой параметр упомянутых классов можно задать как статичное значение (int, float, etc), как функцию или
 как наследника класса Flexible.
 Например, если вы хотите создать инстанс класса аугментации Cutout с варьирующимся параметром `num_holes` от 40 до 50, 
 то вы можете передать функцию или лямбду:
 ```python
cutout_trs = Cutout(num_holes = lambda: np.random.randint(40, 50))
 ```
Если же вы хотите зафиксировать `num_holes` равным 100, просто передайте это значение:
 ```python
cutout_trs = Cutout(num_holes = 100)
 ```
То же самое работает и для генераторов, и для писателей.


Далее подробнее ознакомимся с каждой сущностью поподробнее.

### Писатели
[↑ up](#туториал-по-использованию)

Инстансы этих классов возвращают текст по вызову инстанса. Каждый класс:
 1. наследуется от `BaseWriter`;
 1. вызывает конструктор базового класса, передавая обязательные параметры: цвет текста и шрифт 
 (`PIL.ImageFont.FreeTypeFont`) - имя является опциональным параметром;
 1. реализует метод `write`, возвращающий строку.
 
 Пример писателя серийных номеров в русских паспортах: 
```python
class SerialNumberWriter(BaseWriter):
    def __init__(self, font, color=(0, 0, 0), name=None):
        super().__init__(font=font, name=name, color=color)

    def write(self):
        return f'{random.randint(0, 100):02d}  {random.randint(0, 100):02d}  {random.randint(0, 1000000):06d}'
```

<br>

Пример использования (в переменной `fonts` лежит список из объектов `PIL.ImageFont.FreeTypeFont`, а в `color` tuple из трех 
значений:
```python
serial_writer = SerialNumberWriter(font = lambda: np.random.choice(fonts), color)
text = serial_writer()
print(text)
    > '53 17 890617'  
```

<br>

Например так можно создать нужный шрифт (для этого надо знать путь до желаемого шрифта и задать размер кегля):
```python
from PIL import ImageFont

path_to_font = '/home/ds/tmp/fonts/Arsis.ttf'
font_size = 15

font = ImageFont.truetype(path_to_font, font_size)
```

<br>


Готовые писатели определены в `cftcv.ocr_generator.writers`: 

- `TextWriter` - генерирует текст из случайных комбинаций переданных символов;
- `DateWriter` - генерирует даты в формате DD.MM.YYYY, но можно варьировать формат постпроцессом;
- `ComposeWriter` - принимает список писателей, по вызову инстанса возвращает результат вызова случайного писателя.

### Аугментации
[↑ up](#туториал-по-использованию)

Инстансы этих классов преобразуют изображение `np.ndarray`, которое передается в вызов инстанса. Каждый класс:
 1. наследуется от `BaseTransform`;
 1. реализует метод `transform`, принимающий и возвращающий `np.ndarray`;
 1. вызывает конструктор базового класса, передавая параметры, которые потребуются для реализации функционала класса 
 (если какие-то параметры генерируемые, то есть заданы функцией, тогда они будут автоматически генерироваться перед 
 каждым вызовом инстанса или метода `transform`), а также передавая опциональные вероятность и имя: `p` и `name`.
 
Пример определения класса аугментации: 
```python
class Resize(BaseTransform):
    def __init__(self, size, interpolation = cv2.INTER_NEAREST, name = None, p = 1):
        super().__init__(size=size, interpolation=interpolation, p=p, name=name)

    def transform(self, image: np.ndarray) -> np.ndarray:
        image = cv2.resize(image, self.size, interpolation=self.interpolation)
        return image
```
Пример использования:
```python
resize_trs = Resize(size = lambda: np.random.randint(2,5,2), p = 0.4)
augmented = resize_trs(image)

print(type(image), image.shape)
    > (numpy.ndarray, (64, 1024, 4))

print(type(augmented), augmented.shape)
    > (numpy.ndarray, (64, 1024, 4))
```

Готовые аугментации определены в `cftcv.ocr_generator.ocr_transform`: 

- `Blur` - добавляет блюр (с вероятностью 0.5 используется cv2.blur и с вероятностью 0.5 cv2.gaussianblur); 

- `BluredCutout` - применяет заблюренный cutout (сначала генерирует cutout-маску, затем маска блюрится и применяется); 

- `Brightness` - изменяет яркость изображения;

- `Compose` - композиция transform-ов (по аналогии с albumentations принимает список с аугментациями, которые будут
применятся последовательно при вызове инстанса этого класса);

- `Cutout` - вырезает случайные прямоугольниые области, заполняя их указанным цветом;

- `ElasticTransform` - 
[гибкие искажения картинки](https://www.kaggle.com/bguberfain/elastic-transform-for-data-augmentation);

- `Erode` - морфологическое преобразование: эрозия;

- `MultByBluredNormal` - умножает попиксельно изображение на маску из нормального распределения
 (маска клипается к значениям [0,1], размер маски = размер изображения / scale, 
 после получения маски она приводится с помощью cv2.resize к размеру изображения);

- `Pepper` - добавляет (сыплет) перец;

- `Perspective` - искажение перспективы (изменение угла обзора);

- `RGB2GRAY` - конвертирует изображение в gray с помощью cv2.cvtcolor() 
(если save_channels == true, возвращается 3-канальное изображение (все каналы одинаковы), 
иначе матрица без каналов);

- `ReduceSizeIfNeeded` - сжимает картинку до размера <= указанному, пытаясь сохранить соотношение сторон;

- `Resize` - приводит картинку к требуемому размеру с помощью cv2.resize();

- `ResizeAndPad` - приводит картинку по высоте к dst_size[0], 
стараясь сохранить соотношение сторон (если после приведения ширина больше dst_size[1], 
тогда приводим к размеру dst_size. если ширина меньше dst_size[1], 
то добавляем падинги по ширине, чтобы ширина соответствовала dst_size[1],
размер левого падинга выбирается случайно, размер правого вычисляется по остаточному принципу);

- `ResizeByStacking` - изменяет размер картинки за счет ее повторения по осям x и y;

- `Rotate` - случайно поворачивает на угол angle;

- `Scale` - приводит картинку размером (h,w) к размеру (scale*h,scale*w);

- `ScaleUnscale` - масштабирует картинку размером (h,w) к размеру (scale*h,scale*w) и обратно;

- `Speckle` - создает эффект шероховатости;

- `Warping` - сжимает/растягивает изображение по горизонтали;


### Генераторы
[↑ up](#туториал-по-использованию)

Инстансы этих классов возвращают 3 объекта: картинку (np.ndarray, H x W x C), маску (np.ndarray, H x W) и значение 
по вызову инстанса. В частном случае значением может быть отображаемый на изображении текст. 
Маска - матрица, обозначающая фон нулевыми пикселями и отрендеренное значение пикселями отличными от нуля  

Каждый класс:
 1. наследуется от `BaseGenerator`;
 1. реализует методы: `random_value` - возвращающий значение (например, текст) и `render` - принимающий значение из 
 первого метода и возвращающий картинку с маской - обе типа `np.ndarray`;
 1. вызывает конструктор базового класса, передавая параметры, которые потребуются для реализации функционала класса 
 (если какие-то параметры генерируемые, то есть заданы функцией, тогда они будут автоматически генерироваться перед 
 каждым вызовом инстанса или метода `render`), а также передавая опциональные вероятность и имя: `p` и `name`.

Готовые генераторы определены в `cftcv.ocr_generator.generators`, ознакомимся ближе с каждым.

#### BackgroundGenerator
[↑ up](#туториал-по-использованию)

```python
class BackgroundGenerator(BaseGenerator):
    """Генератор изображений (фоновых картинок).

    Сгенерировать картинку можно по индексу (для совместимости с torch.utils.data.DataLoader)
    или вызвав экземпляр класса. Любой из способов возвращает картинку и маску.
    Высота и ширина маски совпадает с высотой и шириной картинки. Маска заполнена значением alpha.

    :param image: картинка, которую будет возвращать генератор
    :param transforms: трансформации, которые будут применены к 4-х канальной картинке: rgb + alpha (default None)
    :param name: имя объекта, если None, тогда именем будет имя класса в нижнем регистре (default None)
    :param alpha: значение, которым будет заполнена маска (default 255)
    :param size: размер (для совместимости с torch.utils.data.DataLoader) (default 1000)
    """

    def __init__(self, image: Union[np.ndarray, Callable[[], np.ndarray]], transforms: Optional[BaseTransform] = None, 
        name: Optional[str] = None, alpha: Union[int, Callable[[], int]] = 255, size: int = 1000) -> NoReturn:
```

Данный генератор применяет аугментации к картинке `image` и возвращает ее по индексу
или по вызову инстанса вместе с маской и значением. Маска заполнена нулями, вместо значения возвращается None. Важно
отметить, что если вы хотите "генерировать" разные картинки случайным образом, тогда параметр `image` должен быть
генерируемым параметром (функцией или ламбдой, возвращающей нужную картинку).

<br>

Пример использования (`images` - список с картинками состоящий из объектов `np.ndarray`):
```python
back_gen = BackgroundGenerator(image = lambda: np.random.choice(images))
image, mask, value = back_gen()
```

#### Barcode128Generator
[↑ up](#туториал-по-использованию)

```python
class Barcode128Generator(BaseGenerator):
    """Генератор штрих-кодов с кодировкой 128A, 128B или 128С.

    Сгенерировать картинку можно по индексу (для совместимости с torch.utils.data.DataLoader)
    или вызвав экземпляр класса. Любой из способов возвращает картинку, маску штрихов баркода и закодированное значение.

    :param writer: писатель, который генерирует серийный номер по вызову
    :param transforms: трансформации, которые будут применены к 4-х канальной картинке (rgb + alpha) (default None)
    :param top_pad: отступ сверху при генерации штрихов (default 0)
    :param bot_pad: отступ снизу при генерации штрихов (default 0)
    :param line_width: ширина одного штриха (default 0.2)
    :param line_height: высота одного штриха (default 5)
    :param name: имя объекта, если None, тогда именем будет имя класса в нижнем регистре (default None)
    :param size: размер (для совместимости с torch.utils.data.DataLoader) (default 1000)
    """

    def __init__(self, writer: Union[BaseWriter, Callable[[], BaseWriter]], transforms: Optional[BaseTransform] = None,
            top_pad: Union[float, Callable[[], float]] = 0, bot_pad: Union[float, Callable[[], float]] = 0,
            line_width: Union[float, Callable[[], float]] = 0.2, line_height: Union[float, Callable[[], float]] = 5,
            name: Optional[str] = None, size: int = 1000
    ) -> NoReturn:
```

Данный генератор использует писателя (или композицию писателей если это инстанс класса `ComposeWriter`) для генерации 
серийного номера, после чего отображает этот номер в штрихкод типа [128B](https://en.wikipedia.org/wiki/Code_128).

<br>

Пример использования (`writer` - объект наследника BaseWriter, генерирующий строки с серийным номером):
```python
class SerialWriter(BaseWriter):
    def __init__(self, length):
        self.length = length
        super().__init__(length=length)
    
    def write(self):
        value = ''.join(['%02d' % np.random.randint(100) for _ in range(self.length)])
        return value

bar_gen = Barcode128Generator(writer=SerialWriter(10), top_pad = 0, bot_pad = 0)
image, mask, value = bar_gen()
```


#### TextGenerator
[↑ up](#туториал-по-использованию)

```python
class TextGenerator(BaseGenerator):
    """Генератор изображений с текстом.

    Сгенерировать картинку можно по индексу (для совместимости с torch.utils.data.DataLoader)
    или вызвав экземпляр класса. Любой из способов возвращает картинку, маску текста и текст.

    :param writer: писатель, который генерирует текст по вызову
    :param transforms: трансформации, которые будут применены к 4-х канальной картинке (rgb + alpha) (default None)
    :param spacing: среднее значение расстояния между буквами (default 0.7)
    :param sp_var: параметр, отвечающий за вариативность расстояния между буквами расстояние - случайная (default 0.05)
    величина из равномерного распределения Uni[spacing - sp_var * font_size, spacing + sp_var * font_size],
    где font_size - размер шрифта, диктуемый writer-ом
    :param alpha: значение, которым будет заполнена маска в месте букв (default 255)
    :param pad: отступ от каждого края при рендере текста (в пикселях) (default 255)
    :param name: имя объекта, если None, тогда именем будет имя класса в нижнем регистре (default None)
    :param size: размер (для совместимости с torch.utils.data.DataLoader) (default 1000)
    """

    def __init__(self, writer: Union[BaseWriter, Callable[[], BaseWriter]], transforms: Optional[BaseTransform] = None,
        spacing: Union[float, Callable[[], float]] = 0.7, sp_var: Union[float, Callable[[], float]] = 0.05,
        alpha: Union[int, Callable[[], int]] = 255, pad: int = 1, name: Optional[str] = None, size: int = 1000,
    ) -> NoReturn:
```

Данный генератор использует писателя (или композицию писателей если это инстанс класса `ComposeWriter`) для генерации 
текста, после чего рендерит текст цветом, указанным в писателе поверх фона такого же цвета (это нужно для того, 
чтобы не происходило размитие текста с фоном, в следствие чего может искажаться цвет краев букв: представьте что вы 
сгенерили черный текст поверх белого фона, заблюрили картинку с текстом и маску - во время блюра картинки, края черных 
букв смешаются с белым фоном, что может являться неожиданным поведением).

<br>

Пример использования (`images` - список с картинками состоящий из объектов `np.ndarray`):
```python
date_writer = DateWriter(font, color)

text_gen = TextGenerator(date_writer)
image, mask, text = text_gen()
```


#### OverlayGenerator
[↑ up](#туториал-по-использованию)

```python
class OverlayGenerator(BaseGenerator):
    """
    Генератор наложения.

    Генерирует фоновую картинку генератором background_gen,
    генерирует картинку и маску переднего плана генератором foreground_gen,
    затем накладывает передний план поверх фона, с учетом маски (альфа-канала).
    Сгенерировать картинку можно по индексу (для совместимости с torch.utils.data.DataLoader)
    или вызвав экземпляр класса. Любой из способов возвращает картинку, маску и текст.

    :param background_gen: генератор фона
    :param foreground_gen: генератор переднего плана
    :param constraints: функция, преобразующая словарь параметров  (default None)
    :param pad: отступы от каждой стороны фона при наложении текста (default (0, 0, 0, 0))
    :param transforms: трансформации, которые будут применены к 4-х канальной картинке (rgb + alpha) (default None)
    :param alpha: значение, которым будет заполнена маска (default 255)
    :param fit_mode: указывает что ресайзим перед наложением "background" или "foreground", (default None)
    если None, тогда наслоение будет происходить в случайную область фона
    :param name: имя объекта, если None, тогда именем будет имя класса в нижнем регистре (default None)
    :param size: размер (для совместимости с torch.utils.data.DataLoader) (default 1000)
    """

    def __init__(
        self,
        background_gen: BaseGenerator,
        foreground_gen: BaseGenerator,
        constraints: Optional[types.FunctionType] = None,
        pad: Union[Tuple[int, int, int, int], Callable[[], Tuple[int, int, int, int]]] = (0, 0, 0, 0),
        transforms: Optional[BaseTransform] = None,
        alpha: Union[int, Callable[[], int]] = 255,
        fit_mode : Optional[str] = 'background',
        name: Optional[str] = None,
        size: int = 1000,
    ) -> NoReturn:
```

Данный генератор накладывает изображение генератора переднего плана `foreground_gen` поверх генератора фона 
`background_gen`, используя маску переднего плана. Если значение маски в каком-либо пикселе равно 0, тогда в итоговом
изображении на этом месте будет пиксель фона, если 255, то пиксель переднего плана, в иных случаях - взвешанная сумма.


Важно подметить: генератором фона может быть другой инстанс класса OverlayGenerator.

<br>

Пример использования ():
```python
# определяем генератор фоновых изображений
back_gen = BackgroundGenerator(image = lambda: np.random.choice(images))

# определяем генератор текста
date_writer = DateWriter(font, color)
text_gen = TextGenerator(date_writer)

# определяем наслаивающий генератор
over_gen = OverlayGenerator(back_gen, text_gen)

# генерируем картинку с текстом поверх фона, маску текста и отображенный текст
image, mask, text = over_gen()
```


### Регулирование параметров генерации с помощью constraints
[↑ up](#туториал-по-использованию)

Этот функционал еще обновится чуть позже, ибо его надо немного упростить, но он уже есть и суть в том, 
что через параметр `constraints` в конструкторе `OverlayGenerator` можно задавать функцию, которая принимает все 
возможные параметры от всех объектов, участвующих в генерации картинки: писателей, аугментаций, генераторов - то есть
от всех инстансов класса `Flexible`. Это и есть основная фича. Переданная функция должна принимать и возвращать один
параметр: словарь. Попробуйте создать свой генератор, передав в `OverlayGenerator` следующую функцию как параметр
`constraints`, и вы увидите структуру словаря:

```python
def constraints(params):
    from pprint import pprint
    pprint(params)
    return params
```

### Зоопарк настроенных генераторов
[↑ up](#туториал-по-использованию)

В пакете `cftcv.ocr_generator.zoo` лежат уже настроенные генераторы (на 14.05.2020 два генератора:
`frida.py` и `passport_rf.py`). Если вы хотите сохранить свой настроенный генератор в библиотеку `cftcv`, тогда 
вам следует сохранить "сборку" генератора в виде Python-скрипта (смотрите на структуру уже созданных сборок) 
и поместить его в `cftcv.ocr_generator.zoo`. 

Также есть возможность сохранить статичные данные (фоны и шрифты) на s3, чтобы не потерять их. Подробнее об этом описано
[ниже](#хранение-шрифтов-и-фонов-на-s3).

#### Примеры использования настроенных генераторов

1. чтобы получить генератор полей для проекта FRIDA (распознавание платежных документов):
```python
from cftcv.ocr_generator.zoo import frida

vocab = frida.vocab
gen = frida.get_generator_v2(vocab=vocab, n=5000, str_padding_to=45)

image, mask, target = gen()
```
`vocab` - вокабуляр, который будет использоваться для перевода символов в числа (в порядковый номер символа в `vocab`)
`n` - длинна генератора (требуется для совместимости с `torch.utils.data.DataLoader`)
`str_padding_to` - до какого длинны надо заполнить таргет нулями справа

2. пример загрузки генератора полей из паспортов РФ:
```python
from cftcv.ocr_generator.zoo import passport_rf

vocab = list("ЁЙЦУКЕНГШЩЗХЪФЫВАПРОЛДЖЭЯЧСМИТЬБЮ()/.-№1234567890 ")
gen = passport_rf.get_generator_v2(vocab=vocab, n=5000, str_padding_to=45)
```
`vocab` - вокабуляр, который будет использоваться для перевода символов в числа (в порядковый номер символа в `vocab`)
`n` - длинна генератора (требуется для совместимости с `torch.utils.data.DataLoader`)
`str_padding_to` - до какого длинны надо заполнить таргет нулями справа


### Как создать свои шрифты или фоны

#### Шрифты
[↑ up](#туториал-по-использованию)

Большой выбор шрифтов с удобной фильтрацией представлен 
[здесь](https://www.ph4.ru/fonts_fonts.php).<br>

А создать нужный шрифт можно следующим образом (для этого надо знать путь до желаемого шрифта и задать размер кегля):
```python
from PIL import ImageFont

path_to_font = '/home/ds/tmp/fonts/Arsis.ttf'
font_size = 15

font = ImageFont.truetype(path_to_font, font_size)
```

#### Фоны
[↑ up](#туториал-по-использованию)

Фоновые картинки можно быстро нарезать с помощью [VIA](http://www.robots.ox.ac.uk/~vgg/software/via/).
Требуется лишь загрузить в VIA оригинальные картинки и выделить прямоугольниками области, которые могут
послужить фонами для генерируемых полей. Затем потребуется нарезать эти прямоугольники с помощью скрипта и сложить
как картинки в какую-либо папку, после чего их можно [сохранить на s3](##загрузка-шрифтов-и-фонов-на-s3), чтобы не потерять.


todo: добавить код, который нарезает кропы из json-конфигурации проекта в VIA и оригинальных картинок.


#### Загрузка шрифтов и фонов на s3
[↑ up](#туториал-по-использованию)

Шрифты и фоновые картинки хранятся на s3 в виде zip-архивов (для каждой задачи отдельный
zip-архив под шрифты и отдельный под картинки). 
Чтобы добавить на s3 новый архив, потребуется скачать AWS CLI, 
с помощью которого можно заливать на s3 файлы через терминал.

##### 1. Установим AWS CLI (без sudo)
[ссылка на оригинальный туториал](https://docs.aws.amazon.com/cli/latest/userguide/install-bundle.html#install-bundle-user)

Выполняем:
```
$ curl "https://s3.amazonaws.com/aws-cli/awscli-bundle.zip" -o "awscli-bundle.zip"
$ unzip awscli-bundle.zip
$ ./awscli-bundle/install -b ~/bin/aws
```
установится в `~/.local/lib/aws` и создастся symlink `~/bin/aws`. 
Если вывод следующей команды является пустым:
```
$ echo $PATH | grep ~/bin
```
тогда требуется выполнить:
```
export PATH=~/bin:$PATH
```

##### 2. Экспортируем переменные окружения 
[ссылка на вирго](https://git.ftc.ru/ml/research/volkov-e.a/dvc-testin) - отсюда берем секретный ключ
```
export AWS_ACCESS_KEY_ID="ml-dvc"
export AWS_SECRET_ACCESS_KEY="PzPu7vW9Wl64nWgTrfXr0Pio"
```
ключ `PzPu7vW9Wl64nWgTrfXr0Pio` актуален на 06.05.2020

##### 3. Сжимаем данные в zip-архив
```
zip -r -j ZIP_PATH DIR_WITH_FILES
```
`DIR_WITH_FILES` - директория, содержащая файлы, которые требуется сжать в единый
zip-архив

`ZIP_PATH` - путь, куда сохранится zip (например `/home/tmp/archive.zip`); старайтесь
давать архивам названия, отражающие задачу, для которой были собраны эти
картинки или шрифты (например не `archive.zip`, а `passport_rf.zip`) 

##### 4. Копируем на s3 
```
http_proxy= aws s3 cp ZIP_PATH s3://ml-dvc/libraries/cftcv/ocr_generator/DIR/FILE_NAME --endpoint-url http://dpcloud01-et.ftc.ru:31900
```
`ZIP_PATH` - путь до zip

`DIR` - либо `backgrounds`, либо `fonts`, в зависимости от того,
что хотите сохранить: картинки или шрифты

`FILE_NAME` - имя архива (последняя часть пути `ZIP_PATH`)


##### 5. Если залили не то и хотите удалить
```
http_proxy= aws s3 rm s3://ml-dvc/libraries/cftcv/ocr_generator/PATH --endpoint-url http://dpcloud01-et.ftc.ru:31900
```
`PATH` - дальнейший путь к тому, что требуется удалить из s3


#### Как забрать шрифты и фоны с s3
[↑ up](#туториал-по-использованию)

Шрифты и фоны загружаются отдельно друг от друга. Чтобы загрузить требуемый архив, требуется знать его название, 
извлечение файлов в `~/.cache/cftcv` произойдет автоматически. 

Пример загрузки шрифтов и фонов для проекта FRIDA:
```python
backgrounds = static.load_backgrounds('frida.zip')
fonts = static.load_fonts('frida.zip', sizes=[14, 15, 16])
```
`sizes` - размеры кегля (возвращаемые шрифты - декартово произведение размеров кегля на шрифты в архиве)

Чтобы вывести список хранящихся на s3 файлах можно воспользоваться функцией `list_s3()`:
```python
static.list_s3()

    >['backgrounds/frida.zip',
    >'backgrounds/passport_rf.zip',
    >'fonts/frida.zip',
    >'fonts/passport_rf_default.zip',
    >'fonts/passport_rf_serial.zip']
```


### Пример сборки генератора
[↑ up](#туториал-по-использованию)

Ниже представлен пример сборки жизнеспособного генератора:
```python
# импортируем базовый класс писателя, чтобы определить писателя серийных номеров
from cftcv.ocr_generator.base import BaseWriter

# импортируем писателей, аугментации и генераторы
from cftcv.ocr_generator.generators import BackgroundGenerator, TextGenerator, OverlayGenerator
from cftcv.ocr_generator.writers import TextWriter, ComposeWriter
from cftcv.ocr_generator.ocr_transform import (Compose, Cutout, Perspective, Blur, ElasticTransform,
                                               Warping, Brightness, Rotate, MultByBluredNormal, 
                                               BluredCutout, ScaleUnscale, Perspective, ResizeAndPad)


# иные библиотеки
from glob import glob
import numpy as np
from numpy.random import choice, randint, uniform
from PIL import Image, ImageFont

    
    


# 1) Подготовим писателей

# загрузим шрифты
to_fonts = glob('/home/ds/tmp/static_data_for_tutorial/fonts/*')
sizes = [10, 15, 20]
fonts = list(ImageFont.truetype(f, s) for f, s in zip(to_fonts, sizes))

# определим свой класс и инстанс писателя серийных номеров, состоящих из 10 цифр
class SerialWriter(BaseWriter):
    def __init__(self, font, color=(0, 0, 0), name=None):
        super().__init__(font=font, name=name, color=color)

    def write(self):
        number = [str(randint(1, 10))]
        for _ in range(10):
            number.append(str(randint(1, 10)))
        return ''.join(number)

serial_writer = SerialWriter(font = lambda: choice(fonts),
                             color = (100, 100, 100))

# определим писателя текста и объединим писателей в композицию
text_writer = TextWriter(font = lambda: choice(fonts),
                         color = (100, 100, 100), 
                         vocab = list('qwertyuiopasdfghjklzxcvbnm.,'),
                         word_len_lim = (5, 10),
                         words_count_lim = (2, 4))
compose_writer = ComposeWriter([text_writer, serial_writer])
    
    

# 2) Подготовим генератор фона

# создадим список с картинками-бэкграундами
to_backgrounds = glob('/home/ds/tmp/static_data_for_tutorial/backgrounds/*')
backgrounds = list(np.asarray(Image.open(x)) for x in to_backgrounds)

# подготовим аугментации
trs_back = Compose(
    [
        Warping(warp_coeff=lambda: uniform(1.5, 2.4), p=0.5),
        Brightness(brightness=lambda: randint(50, 250), p=0.2)
    ]
)

# определим генератор
back_gen = BackgroundGenerator(image = lambda: choice(backgrounds), transforms=trs_back)



    
    
# 3) Подготовим генератор текста

trs_text = Compose(
    [
        Rotate(lambda: uniform(-2, 2), p=1, name='text_rotate'),
        MultByBluredNormal(channels=3, std=0.5, scale=1.5, p=0.1),
        BluredCutout(
            size=lambda: randint(2, 12, 2),
            num_holes=lambda: randint(5, 30),
            channels=[0, 1, 2],
            ksize=21,
            p=0.5,
        ),
        Warping(warp_coeff=lambda: uniform(0.7, 2.4), p=0.2),
        ScaleUnscale(scale=lambda: uniform(0.6, 0.9), interpolation=1, p=0.3),
        Perspective(scale=(5, 3), p=0.15),
    ]
)

text_gen = TextGenerator(
    writer = compose_writer,
    spacing = lambda: uniform(-0.2, 0.2),
    sp_var = 0.08,
    pad = 1,
    alpha = 255,
    transforms = trs_text,
    size = 5000,
)



    
    
# 4) Подготовим наслаивающий генератор
    
trs_over = Compose(
    [
        Blur(ksize=lambda: choice([3]), p=0.2),
        ScaleUnscale(scale=lambda: uniform(0.8, 0.95), p=0.3),
        Brightness(brightness=lambda: randint(0, 120), p=0.15),
        
        ResizeAndPad((32, 512)), # приводим картинки по высоте к 32, ширину дополняем падингами
        
        ElasticTransform(alpha=randint(500, 800), sigma=randint(8, 10), p=0.1),
    ]
)
    
    
overlay_gen = OverlayGenerator(
    background_gen = back_gen,
    foreground_gen = text_gen,
    transforms = trs_over,
    pad = lambda: [randint(0, 5), randint(0, 5), randint(0, 5), randint(0, 5)],
    fit_mode = 'background',
    name = "over_gen",
    size = 5000,
)



    
    
# 5) Генерируем картинки
    
for _ in range(5):
    img, mask, text = overlay_gen()
    print(text)
    display(Image.fromarray(img))
```

Результат:

 <img src='./docs/static/images/gen_example.png' alt="gen example">
