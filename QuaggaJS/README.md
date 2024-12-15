# Локализация штрих-кодов в библиотеке QuaggaJS
QuaggaJS - мощная библиотека для локализации и распознавания штрих-кодов. В данном документе содержится объяснение работы локализатора,
используемого в данной библиотеке.

## Как работает локализатор?
Основной функцией для локализации штрих-кодов является `locate` из файла `barcode_locator.js`.
Этапы работы данной функции приведены ниже:

1. Уменьшение масштаба изображения (опционально)
```js
if (_config.halfSample) {
    halfSample(_inputImageWrapper, _currentImageWrapper);
}
```
   - Если в конфигурации установлен флаг `halfSample`, изображение уменьшается в 2 раза по ширине и высоте
   - Уменьшение масштаба ускоряет обработку на этапе поиска патчей, особенно для больших изображений

2. Бинаризация изображения
```js
binarizeImage();
```
  - Преобразует изображение в бинарный формат при помощи пороговой обработки методом Оцу с одним порогом
  - Получается изображение, где светлые области (потенциальные полосы штрих-кода) выделены на темном фоне

3. Поиск патчей
```js
patchesFound = findPatches();
// return unless 5% or more patches are found
if (patchesFound.length < _numPatches.x * _numPatches.y * 0.05) {
    return null;
}
```
  - Функция `findPatches` разбивает изображение на сетку патчей и анализирует каждый из них на наличие контрастных полос, характерных для штрих-кодов
  - Если найдено менее 5% патчей (в зависимости от размеров сетки `_numPatches.x` и `_numPatches.y`), функция возвращает `null`, так как вероятность
  наличия штрих-кодов слишком мала
  - В результате получается массив патчей (координаты и характеристики), которые могут содержать штрих-коды

4. Растеризация угловой схожести
```js
// rasterrize area by comparing angular similarity;
var maxLabel = rasterizeAngularSimilarity(patchesFound);
if (maxLabel < 1) {
    return null;
}
```
  - Функция `rasterizeAngularSimilarity` группирует патчи на основе схожести их угловых направлений
  - Так как, одномерные штрих-коды представляют собой параллельные линии, патчи с одинаковыми угловыми характеристиками объединяются
  - Если не найдено ни одной группы патчей (`maxLabel < 1`), функция возвращает `null`

5. Поиск наибольших соединенных областей
```js
// search for area with the most patches (biggest connected area)
topLabels = findBiggestConnectedAreas(maxLabel);
if (topLabels.length === 0) {
    return null;
}
```
  - Функция `findBiggestConnectedAreas` определяет крупнейшие области, содержащие наибольшее количество патчей
  - Большие области с высокой плотностью патчей с большей вероятностью содержат штрих-коды
  - Функция отбирает только области, объединяющие 5 или более патчей. Если такие области не найдены, функция возвращает `null`

6. Определение bounding boxes
```js
boxes = findBoxes(topLabels, maxLabel);
return boxes;
```
  - Функция `findBoxes` рассчитывает координаты bounding boxes вокруг найденных областей
  - Возвращает массив объектов с координатами прямоугольников, которые предполагают наличие штрих-кодов

### Реализация функции `locate`
```js
locate: function() {
        var patchesFound,
            topLabels,
            boxes;

        if (_config.halfSample) {
            halfSample(_inputImageWrapper, _currentImageWrapper);
        }

        binarizeImage();
        patchesFound = findPatches();
        // return unless 5% or more patches are found
        if (patchesFound.length < _numPatches.x * _numPatches.y * 0.05) {
            return null;
        }

        // rasterrize area by comparing angular similarity;
        var maxLabel = rasterizeAngularSimilarity(patchesFound);
        if (maxLabel < 1) {
            return null;
        }

        // search for area with the most patches (biggest connected area)
        topLabels = findBiggestConnectedAreas(maxLabel);
        if (topLabels.length === 0) {
            return null;
        }

        boxes = findBoxes(topLabels, maxLabel);
        return boxes;
}
```

## Основные функции
Рассмотрим более подробно используемые при локализации функции из различных файлов библиотеки. Далее будут приведены описания функций, их входные и выходные значения, а также основные этапы их работы.

### barcode_locator.js
1. <b>Функция `init`</b>
    - <b>Описание:</b> Функция для инициализации конфигурации и массива пикселей
    - <b>Входные данные:</b>
         - `inputImageWrapper` - обертка для входного изображения, предоставляющая доступ к пикселям изображения и его размерам
        - `config` - объект конфигурации, содержащий настройки алгоритма локализации
    - <b>Процесс:</b>
        - Конфигурация и входное изображение сохраняются в глобальные переменные
        - Инициализируются внутренние буферы и структуры данных, необходимые для работы локализатора
        - Настраивает холст для визуализации промежуточных шагов алгоритма (если отладка включена в конфигурации) 
    - <b>Код функции:</b>
    ```js
    init: function(inputImageWrapper, config) {
        _config = config;
        _inputImageWrapper = inputImageWrapper;

        initBuffers();
        initCanvas();
    }
    ```
2. <b>Функция `locate`</b>
   - <b>Описание:</b> Основная функция для локализации штрих-кодов на изображении
   - <b>Входные данные:</b>
       - `_inputImageWrapper` - объект, содержащий данные исходного изображения
   - <b>Выходные данные:</b>
       - `boxes` - массив локазизованных областей с предполагаемым штрих-кодом
   - <b>Процесс:</b>
       - Если в конфигурации задан параметр `halfSample`, то выполняется уменьшение изображения в 2 раза по ширине и высоте
       - Выполняется бинаризация изображения при помощи функции `binarizeImage`
       - Изображение разделяется на сетку патчей, каждый из которых анализируется на содержание штрих-кода. Если найдено менее 5% патчей (в зависимости от
         размеров сетки `_numPatches.x` и `_numPatches.y`), функция возвращает `null`, так как вероятность наличия штрих-кодов слишком мала
       - Выполняется группировка патчей по схожести угловой ориентации (поиск соединенных патчей, имеющих одинаковое направление). В результате получается
         количество меток `maxLabel`, которые соответствуют найденным группам. Если метки отсутствуют `maxLabel < 1`, функция возвращает `null`
       - Выполняется поиск крупных соединенных областей для того, чтобы исключить мелкие или разрозненные области. В результате получается массив крупных
         областей. Если длина данного массива равна 0, функция возвращает `null`
       - Выполняется определение bounding boxes на основе полученных крупных соединенных областей. В результате получается
         массив координат прямоугольников `boxes`
   - <b>Код функции:</b>
   ```js
   locate: function() {
        var patchesFound,
            topLabels,
            boxes;
        
        if (_config.halfSample) {
            halfSample(_inputImageWrapper, _currentImageWrapper);
        }
        
        binarizeImage();
        patchesFound = findPatches();
        // return unless 5% or more patches are found
        if (patchesFound.length < _numPatches.x * _numPatches.y * 0.05) {
            return null;
        }
        
        // rasterrize area by comparing angular similarity;
        var maxLabel = rasterizeAngularSimilarity(patchesFound);
        if (maxLabel < 1) {
            return null;
        }
        
        // search for area with the most patches (biggest connected area)
        topLabels = findBiggestConnectedAreas(maxLabel);
        if (topLabels.length === 0) {
            return null;
        }
        
        boxes = findBoxes(topLabels, maxLabel);
        return boxes;
    }
   ```

### skeletonizer.js

### rasterizer.js

### tracer.js
