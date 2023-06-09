
# Обзор статьи "[Video Face Manipulation Detection through Ensemble of CNNs](https://paperswithcode.com/paper/video-face-manipulation-detection-through)"

## Постановка
Цель исследования заключается в разработке метода для обнаружения манипуляций в видео, связанных с изменением лица человека. Точнее, ставится задача определения, было ли видео подвергнуто цифровой обработке, в результате которой изменяется внешность лица, например, путем накладывания масок на лицо.

## Верхнеуровневое описание
Идея исследования заключается в использовании ансамбля сверточных нейронных сетей (CNN), каждая из которых обучается на различных признаках, связанных с изображением. Ансамбль состоит из четырех нейронных сетей, каждая из которых обучена на одном из четырех различных наборов признаков: на оригинальном видео, на оптическом потоке движения, на кадрах, полученных с помощью метода выравнивания лица, и на масках лица, созданных с помощью стороннего алгоритма. Каждая нейронная сеть в ансамбле использует сверточные слои и полносвязные слои для извлечения признаков и классификации входных видео.

## Детальное описание
Для обучения нейронных сетей был использован датасет Deepfake Detection Challenge (DFDC), состоящий из 50 000 видео со вставленными лицами, созданными с использованием сторонних алгоритмов генерации и обработки изображений. Общий подход к обучению состоит в двух этапах: предобучение и дообучение. На первом этапе все нейронные сети обучаются на наборах данных, не связанных с видео (например, на CIFAR-10). Затем на втором этапе нейронные сети дообучаются на датасете DFDC.Архитектура каждой из нейронных сетей включает в себя сверточные слои и полносвязные слои. 

- *В статье авторы представили несколько уникальных инструментов для обнаружения манипуляций в видео:*

1. `Video-based Semi-Supervised Learning (VSSL)` - это метод, который использует несколько первоначальных наборов данных с различными манипуляциями для обучения модели, а затем модель финетюнится на новых данных, чтобы улучшить ее точность.

2. `Mask R-CNN` - это архитектура модели, используемая для обнаружения объектов на изображениях и видео. Она включает в себя глубокую сверточную нейронную сеть, которая выделяет объекты на изображении, и регрессионную сеть, которая предсказывает координаты границ объектов.

3. `Stacked Hourglass Networks (SHNs)` - это архитектура модели, используемая для обработки изображений с высокой точностью. Она состоит из нескольких параллельных сверточных нейронных сетей, которые обрабатывают изображение в разных масштабах. Каждая сеть генерирует карты точек интереса, которые затем объединяются, чтобы получить более точный результат.

4. `Spatial-Attention Mechanism` - это метод, который позволяет модели сфокусироваться на наиболее важных участках изображения. Он используется для определения области лица на изображении, которую необходимо обработать.

5. `Confidence Map` - это карта, которая показывает, насколько уверена модель в том, что определенный участок изображения содержит лицо. Она используется для определения того, является ли изображение подлинным или подверглось манипуляции.

6. `Preprocessing techniques` - авторы также использовали несколько методов предобработки данных, таких как ресайзинг и центрирование, чтобы обеспечить более стабильное и точное обнаружение манипуляций в видео.

Для обнаружения фреймов, содержащих измененное лицо, используется механизм на основе расстояния Махаланобиса. Он используется для вычисления расстояния между распределением признаков текущего кадра и распределением признаков всех остальных кадров в видео. Для этого вычисляются среднее значение и ковариационная матрица признаков на основе кадров обучающего набора, и затем используются для вычисления расстояния Махаланобиса для каждого кадра в тестовом видео. Критерием для определения, является ли кадр Deepfake, является пороговое значение для расстояния Махаланобиса.

```python
# Расчет расстояния Махаланобиса
class Mahalanobis(nn.Module):
    def __init__(self):
        super(Mahalanobis, self).__init__()

    def forward(self, input1, input2):
        diff = input1 - input2
        md = torch.mm(torch.mm(diff.t(), self.precision), diff)
        return md.diag()

# Вычисление матрицы точности
def calculate_precision_matrix(features, batch_size):
    precision_matrix = torch.zeros(features.size(1), features.size(1)).cuda()
    for i in range(0, features.size(0), batch_size):
        if i + batch_size < features.size(0):
            batch = features[i:i + batch_size]
        else:
            batch = features[i:]
        batch_size = batch.size(0)
        mean = batch.mean(0)
        diff = batch - mean
        precision_matrix += torch.mm(diff.t(), diff) / (batch_size - 1)
    precision_matrix = torch.inverse(precision_matrix + torch.eye(features.size(1)).cuda() * 1e-4)
    return precision_matrix
```

Кроме того, в статье был предложен новый метод на основе декомпозиции матрицы ковариации, который улучшает производительность метода на основе расстояния Махаланобиса и обеспечивает более стабильную работу в условиях, когда обучающие и тестовые данные различаются. Вместо использования единой матрицы ковариации для всех видео в обучающем наборе, каждое видео разбивается на несколько фрагментов, и для каждого фрагмента вычисляется матрица ковариации. Затем для каждого тестового кадра вычисляется расстояние до матриц ковариации каждого фрагмента обучающего видео, и выбирается наименьшее расстояние.

```python
# Расчет расстояния Махаланобиса с использованием разложения ковариационной матрицы
class MahalanobisDecomposed(nn.Module):
    def __init__(self):
        super(MahalanobisDecomposed, self).__init__()

    def forward(self, input1, input2):
        md = torch.zeros((input1.shape[0], input2.shape[0]), dtype=torch.float32).cuda()
        for i in range(input1.shape[0]):
            diff = input1[i] - input2
            md[i] = (diff @ self.precision[i]) * diff.sum(-1)
        return md

# Вычисление матрицы точности для ковариационного разложения
def calculate_precision_matrices(features, batch_size):
    precisions = []
    for i in range(0, features.shape[0], batch_size):
        batch = features[i:i+batch_size]
        cov = torch.matmul(batch.T, batch) / batch.shape[0]
        cov += torch.eye(cov.shape[0]).cuda() * 1e-5
        precision = torch.inverse(cov)
        precisions.append(precision)
    return precisions
```
Для обнаружения Deepfake видео были использованы две архитектуры нейронных сетей: Xception и Inception-ResNet-v2. Эти архитектуры были выбраны, потому что они имеют высокую точность в задаче классификации изображений и могут быть эффективно адаптированы для обнаружения Deepfake видео.

Архитектура Xception включает в себя глубокие сверточные слои с использованием операции разделения по каналам (depthwise separable convolution), которая позволяет значительно уменьшить количество параметров модели. Архитектура Inception-ResNet-v2 сочетает в себе блоки Inception и ResNet и также имеет глубокие сверточные слои с операцией разделения по каналам.

В обоих архитектурах был использован механизм на основе расстояния Махаланобиса для обнаружения Deepfake видео. Для этого была использована функция потерь, основанная на кросс-энтропии, с использованием весов для балансировки классов. Веса были выбраны таким образом, чтобы учитывать относительное количество Deepfake и настоящих видео в датасете.

```python
# Функция потерь на основе кросс-энтропии с весами для балансировки классов
class WeightedCrossEntropyLoss(nn.Module):
    def __init__(self, weight=None, size_average=None, ignore_index=-100, reduce=None, reduction='mean'):
        super(WeightedCrossEntropyLoss, self).__init__()
        self.register_buffer('weight', weight)
        self.reduction = reduction

    def forward(self, input, target):
        return F.cross_entropy(input, target, weight=self.weight, reduction=self.reduction)
```
Для дообучения нейронных сетей на датасете DFDC была использована техника обучения с учителем (supervised learning). Датасет был разделен на обучающую, валидационную и тестовую выборки в соотношении 80/10/10. Для обучения и дообучения нейронных сетей был использован оптимизатор Adam с параметрами по умолчанию и уменьшением скорости обучения в случае отсутствия улучшений на валидационной выборке.

```python
# Инициализация оптимизатора Adam
optimizer = optim.Adam(model.parameters())

# Уменьшение скорости обучения в случае отсутствия улучшений
scheduler = ReduceLROnPlateau(optimizer, 'min', patience=3, factor=0.1)
```
Кроме того, для улучшения результатов были использованы различные техники аугментации данных, такие как случайные повороты, изменение масштаба и яркости изображения, добавление шума и т.д. Эти техники позволяют увеличить разнообразие данных и сделать обучение более устойчивым к различным искажениям.

## Результаты и метрики
![plot](https://raw.githubusercontent.com/akscent/Tlab/main/CV/img/plot.PNG)
*<center>Рис. 1. Распределение баллов для реальных (оранжевый) и поддельных (синий) выборок для каждой пары сетей на наборах данных FF++ (a) и DFDC (b) (Bonettini, N. et all. Video Face Manipulation Detection through Ensembleof Cnns. [[CrossRef](https://ieeexplore.ieee.org/document/9412711)].)</center>*


Для оценки качества предложенного метода авторы использовали метрику **ROC-AUC** на датасетах **Celeb-DF**, **DFDC** и **FaceForensics++**, которые содержат видео с различными манипуляциями над лицами. Результаты экспериментов показали, что предложенный метод превосходит существующие аналоги на всех трех датасетах. В частности, на датасете Celeb-DF метод достигает `ROC-AUC 0.999`, на датасете DFDC - `0.984` и на датасете FaceForensics++ - `0.990`.

## Идеи для улучшения
Предложенный метод для обнаружения манипуляций  с лицами является достаточно эффективным и показывает высокие результаты на всех трех рассмотренных датасетах. Однако, возможны следующие направления улучшения:

1. Обучение с учителем на большем количестве данных. Так как алгоритм основан на обучении с учителем, то использование большего количества данных для обучения может улучшить качество модели. Также стоит рассмотреть возможность расширения датасета и добавление новых типов манипуляций.
2. Использование более сложной модели. Авторы статьи использовали относительно простую сверточную нейронную сеть. Использование более сложных архитектур, например, ResNet или DenseNet, может дать прирост в качестве.
3. Учет контекста. Так как визуальный контекст может влиять на правильность детектирования манипуляций, возможно стоит рассмотреть варианты добавления информации о контексте в модель, например, добавление дополнительных признаков на основе изображения.
4. Разработка метода для обнаружения манипуляций в режиме реального времени. Стоит рассмотреть возможность разработки метода, который бы мог работать в режиме реального времени и детектировать манипуляции над лицами в реальных условиях, например, в потоковых видеоданных. Для этого можно рассмотреть использование более быстрых и легковесных архитектур нейронных сетей, а также использование оптимизированных методов обработки видеоданных, например, методов на основе вычислительного графа. 
5. Также стоит учитывать возможность применения методов для обнаружения манипуляций в различных условиях освещения и качества видео.
