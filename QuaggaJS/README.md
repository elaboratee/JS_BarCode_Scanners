# Локализация штрих-кодов в библиотеке QuaggaJS
Алгоритм локализации состоит из нескольких этапов. Сначала исходное изображение можеть быть уменьшено в два раза по ширине и высоте для ускорения обработки. Затем выполняется бинаризация изображения с помощью метода Оцу для отделения полос штрих-кода от фона. Далее изображение делится на сетку патчей, каждый из которых анализируется. После этого патчи группируются по схожести угловых направлений, так как одномерные штрих-коды состоят из параллельных линий. Затем определяются области, содержащие наибольшее количество патчей, и на основе этих областей строятся bounding boxes, которые определяют расположение штрих-кодов на изображении. Если на каком-либо из этапов не удается обнаружить достаточно данных, локализация заверщается.

## Работа локализатора
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
    - <b>Процесс работы:</b>
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
   - <b>Процесс работы:</b>
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
   
3. <b>Функция `binarizeImage`</b>
    - <b>Описание:</b> Функция для бинаризации изображения при помощи пороговой обработки
    - <b>Входные данные:</b>
        - `_currentImageWrapper` - исходное изображение для бинаризации
    - <b>Выходные данные:</b>
        - `_binaryImageWrapper` - бинаризованное изображение
    - <b>Процесс работы:</b>
        - Выполняется бинаризация изображения при помощи пороговой обработки методом Оцу. Получается изображение,
          где светлые области (потенциальные полосы штрих-кода) выделены на темном фоне
        - Устанавливает значение всех пикселей на границе бинаризованного изображения равным 0 (черный цвет)
    - <b>Код функции:</b>
    ```js
    function binarizeImage() {
        otsuThreshold(_currentImageWrapper, _binaryImageWrapper);
        _binaryImageWrapper.zeroBorder();
        if (ENV.development && _config.debug.showCanvas) {
            _binaryImageWrapper.show(_canvasContainer.dom.binary, 255);
        }
    }
    ```
    
4. <b>Функция `findPatches`</b>
    - <b>Описание:</b> Анализирует изображение, разбивая его на сетку патчей, скелетизирует каждый патч, находит элементы, похожие на полосы штрих-кодов и
      возвращает массив патчей, которые могут содержать штрих-коды
    - <b>Входные данные:</b>
        - `_numPatches` - объект, задающий количество патчей по ширине и высоте изображения
        - `_subImageWrapper` - обертка текущего патча
        - `_skelImageWrapper` - обертка скелетизированного патча
        - `_labelImageWrapper` - обертка для данных о метках
    - <b>Выходные данные:</b>
        - `patchesFound` - массив патчей, содержаших полосы штрих-кода
    - <b>Процесс работы:</b>
        - Каждый из патчей изображения скелетизируется. Для скелетизированного изобраения устанавливается нулевое значение всех пикселей на границе
        - Скелетизированные патчи растеризуются (также выполняется трассировка контуров) для выделения отдельных полос
        - Выполняется расчет моментов (геометрических характеристик) для растеризованных патчей. В результате получается массив объектов,
          содержащих различные характеристики такие, как центроид, угловая ориентация полос и вектор направления
        - Выполняется анализ моментов и на их основе отбираются патчи, содержащие полосы штрих-кода
    - <b>Код функции:</b>
    ```js
    function findPatches() {
        var i,
            j,
            x,
            y,
            moments,
            patchesFound = [],
            rasterizer,
            rasterResult,
            patch;
        for (i = 0; i < _numPatches.x; i++) {
            for (j = 0; j < _numPatches.y; j++) {
                x = _subImageWrapper.size.x * i;
                y = _subImageWrapper.size.y * j;
    
                // seperate parts
                skeletonize(x, y);
    
                // Rasterize, find individual bars
                _skelImageWrapper.zeroBorder();
                ArrayHelper.init(_labelImageWrapper.data, 0);
                rasterizer = Rasterizer.create(_skelImageWrapper, _labelImageWrapper);
                rasterResult = rasterizer.rasterize(0);
    
                if (ENV.development && _config.debug.showLabels) {
                    _labelImageWrapper.overlay(_canvasContainer.dom.binary, Math.floor(360 / rasterResult.count),
                        {x: x, y: y});
                }
    
                // calculate moments from the skeletonized patch
                moments = _labelImageWrapper.moments(rasterResult.count);
    
                // extract eligible patches
                patchesFound = patchesFound.concat(describePatch(moments, [i, j], x, y));
            }
        }
    
        if (ENV.development && _config.debug.showFoundPatches) {
            for ( i = 0; i < patchesFound.length; i++) {
                patch = patchesFound[i];
                ImageDebug.drawRect(patch.pos, _subImageWrapper.size, _canvasContainer.ctx.binary,
                    {color: "#99ff00", lineWidth: 2});
            }
        }
    
        return patchesFound;
    }
    ```
    
5. <b>Функция `rasterizeAngularSimilarity`</b>
    - <b>Описание:</b> Группирует патчи на основе схожести угловой ориентации (ищет соединенные патчи, имеющие схожее направление)
    - <b>Входные данные:</b>
        - `patchesFound` - массив патчей, содержащих полосых штрих кодов
    - <b>Выходные данные:</b>
        - `label` - количество найденных областей (групп патчей)
    - <b>Процесс работы:</b>
        - Пока существуют необработанные патчи для каждого нового патчка увеличивается метка `label` и выполняется трассировка через `trace`
        - Трассировка выполняется следующим образом:
            - Для каждого патчка совершается обход соседних патчей
            - Если соседний патч пустой, он пропускается
            - Иначе, если схожесть между направлениями патчей, больше порога, рекурсивно вызывается функция трассировки для нового патча
    - <b>Код функции:</b>
    ```js
    function rasterizeAngularSimilarity(patchesFound) {
        var label = 0,
            threshold = 0.95,
            currIdx = 0,
            j,
            patch,
            hsv = [0, 1, 1],
            rgb = [0, 0, 0];
    
        function notYetProcessed() {
            var i;
            for ( i = 0; i < _patchLabelGrid.data.length; i++) {
                if (_patchLabelGrid.data[i] === 0 && _patchGrid.data[i] === 1) {
                    return i;
                }
            }
            return _patchLabelGrid.length;
        }
    
        function trace(currentIdx) {
            var x,
                y,
                currentPatch,
                idx,
                dir,
                current = {
                    x: currentIdx % _patchLabelGrid.size.x,
                    y: (currentIdx / _patchLabelGrid.size.x) | 0
                },
                similarity;
    
            if (currentIdx < _patchLabelGrid.data.length) {
                currentPatch = _imageToPatchGrid.data[currentIdx];
                // assign label
                _patchLabelGrid.data[currentIdx] = label;
                for ( dir = 0; dir < Tracer.searchDirections.length; dir++) {
                    y = current.y + Tracer.searchDirections[dir][0];
                    x = current.x + Tracer.searchDirections[dir][1];
                    idx = y * _patchLabelGrid.size.x + x;
    
                    // continue if patch empty
                    if (_patchGrid.data[idx] === 0) {
                        _patchLabelGrid.data[idx] = Number.MAX_VALUE;
                        continue;
                    }
    
                    if (_patchLabelGrid.data[idx] === 0) {
                        similarity = Math.abs(vec2.dot(_imageToPatchGrid.data[idx].vec, currentPatch.vec));
                        if (similarity > threshold) {
                            trace(idx);
                        }
                    }
                }
            }
        }
    
        // prepare for finding the right patches
        ArrayHelper.init(_patchGrid.data, 0);
        ArrayHelper.init(_patchLabelGrid.data, 0);
        ArrayHelper.init(_imageToPatchGrid.data, null);
    
        for ( j = 0; j < patchesFound.length; j++) {
            patch = patchesFound[j];
            _imageToPatchGrid.data[patch.index] = patch;
            _patchGrid.data[patch.index] = 1;
        }
    
        // rasterize the patches found to determine area
        _patchGrid.zeroBorder();
    
        while (( currIdx = notYetProcessed()) < _patchLabelGrid.data.length) {
            label++;
            trace(currIdx);
        }
    
        // draw patch-labels if requested
        if (ENV.development && _config.debug.showPatchLabels) {
            for ( j = 0; j < _patchLabelGrid.data.length; j++) {
                if (_patchLabelGrid.data[j] > 0 && _patchLabelGrid.data[j] <= label) {
                    patch = _imageToPatchGrid.data[j];
                    hsv[0] = (_patchLabelGrid.data[j] / (label + 1)) * 360;
                    hsv2rgb(hsv, rgb);
                    ImageDebug.drawRect(patch.pos, _subImageWrapper.size, _canvasContainer.ctx.binary,
                        {color: "rgb(" + rgb.join(",") + ")", lineWidth: 2});
                }
            }
        }
    
        return label;
    }
    ```
    
6. <b>Функция `findBiggestConnectedAreas`</b>
    - <b>Описание:</b> Выполняет поиск наиболее крупных соединенных областей патчей. Анализирует сетку меток патчей и возвращает только области,
      содержащие достаточно патчей, чтобы потенациально представлять штрих-код
    - <b>Входные данные:</b>
        - `maxLabel` - общее количество меток (каждая метка представляет уникальную группу патчей)
        - `_patchLabelGrid` - сетка, в которой каждый патч связан с определенной меткой (или областью)
    - <b>Выходные данные:</b>
        - `topLabels` - массив объектов, представляющих наиболее крупные области патчей
    - <b>Процесс работы:</b>
        - Выполняется подсчет патчей для каждой метки
        - Выполняется преобразование массива меток в массив объектов, содержащих метку и количество патчей для данной метки
        - Выполняется сортировка областей в порядке убывания количества патчей
        - Отбираются только те области, которые содержат не менее 5 патчей
    - <b>Код функции:</b>
    ```js
    function findBiggestConnectedAreas(maxLabel){
        var i,
            sum,
            labelHist = [],
            topLabels = [];
    
        for ( i = 0; i < maxLabel; i++) {
            labelHist.push(0);
        }
        sum = _patchLabelGrid.data.length;
        while (sum--) {
            if (_patchLabelGrid.data[sum] > 0) {
                labelHist[_patchLabelGrid.data[sum] - 1]++;
            }
        }
    
        labelHist = labelHist.map(function(val, idx) {
            return {
                val: val,
                label: idx + 1
            };
        });
    
        labelHist.sort(function(a, b) {
            return b.val - a.val;
        });
    
        // extract top areas with at least 6 patches present
        topLabels = labelHist.filter(function(el) {
            return el.val >= 5;
        });
    
        return topLabels;
    }
    ```

7. <b>Функция `findBoxes`</b>
    - <b>Описание:</b> Создает ограничивающие прямоугольники (bounding boxes) для наибольших областей, сгруппированных по угловой ориентации
    - <b>Входные данные:</b>
        - `topLabels` - наиболее крупные области патчей
        - `maxLabel` - количество найденных областей (групп патчей)
    - <b>Выходные данные:</b>
        - `boxes` - массив координат ограничивающих прямоугольников
    - <b>Процесс работы:</b>
        - Для каждой области из `topLabels` производится обход всей сетки патчей и совершается отбор патчей, принадлежащих текущей области
        - Для собранных патчей рассчитывается минимальный ограничивающий прямоугольник, охватывающий все патчи
        - Если прямоугольник успешно создан, он добавляется в выходной массив
    - <b>Код функции:</b>
    ```js
    function findBoxes(topLabels, maxLabel) {
        var i,
            j,
            sum,
            patches = [],
            patch,
            box,
            boxes = [],
            hsv = [0, 1, 1],
            rgb = [0, 0, 0];
    
        for ( i = 0; i < topLabels.length; i++) {
            sum = _patchLabelGrid.data.length;
            patches.length = 0;
            while (sum--) {
                if (_patchLabelGrid.data[sum] === topLabels[i].label) {
                    patch = _imageToPatchGrid.data[sum];
                    patches.push(patch);
                }
            }
            box = boxFromPatches(patches);
            if (box) {
                boxes.push(box);
    
                // draw patch-labels if requested
                if (ENV.development && _config.debug.showRemainingPatchLabels) {
                    for ( j = 0; j < patches.length; j++) {
                        patch = patches[j];
                        hsv[0] = (topLabels[i].label / (maxLabel + 1)) * 360;
                        hsv2rgb(hsv, rgb);
                        ImageDebug.drawRect(patch.pos, _subImageWrapper.size, _canvasContainer.ctx.binary,
                            {color: "rgb(" + rgb.join(",") + ")", lineWidth: 2});
                    }
                }
            }
        }
        return boxes;
    }
    ```
