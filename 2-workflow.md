#### типичный процесс работы с торчом
1. подготовка данных (60-80% данных на обучение, 10-20 на валидацию, остальное 10-20 тест)
2. построение модели ()
3. обучение на данных
4. предсказание (предикт) - ```torch.inference_mode()``` тоже самое что ```torch.no_grad()``` (```no_grad()``` в старом торче)
    - inference_mode() - вырубает градиентное отслеживание парамтеров, для инференсе эту не нужно (для обучения да), короче можно быстро увидеть предикт
```python
with torch.inference_mode(): 
    y_preds = model_0(X_test)
```
5. сохранить модель

#### 1 подготовка данных
- ну тут все тривиально train, test делаем фреймы

#### 2 построение модели
- в торче 4 параметра с которыми можно создать нейронку
- ```torch.nn``` 
  - блоки для строительнства вычисления графов
- ```torch.nn.Parameter```
  - состоит из тензоров их можно юзать с ```nn.Module```, есть параметр ```requires_grad=True``` чтобы обновлять параметры модели с помощзью градиентного спуска
  - в нем веса  или смещение
- ```torch.nn.Module```
  - базовый класс для всех нейронок, все куски нейронки это эти подклассы, если нейронка на торче то она состоит из ```nn.Module``` и у нее есть метод ```forward()```
  - содержит большие блоки = слои
- ```torch.optim``` 
  - в нем разные оптимизационные алгоритмы, помогает параметр в ```nn.Parameter``` чтобы улучшить градиентный спуск
- ```def forward()```	
  - все подклассы от nn.Module должны иметь этот метод, определяет порядок расчета 

#### 3 обучение модели на данных
- большую часть времени ты не будешь знать идеальные параметры для модели :)
- функция потерь
  - функция потерь показывает как хорошо модель предсказывает реальное значение, и чтобы норм модель была надо эту функции потерь минимизировать
  - в торче много функций потерь, самые основные:
    - в торче они тут torch.nn.
    - средняя абсолютная ошибка - усреднённая сумма модулей разницы между реальным и предсказанным значениями (torch.nn.L1Loss()) - менее чувствительна к выбросам. 
    - кросс-энтропия (или логарифмическая функция потерь – log loss) - Кросс-энтропия измеряет расхождение между двумя вероятностными распределениями. Если кросс-энтропия велика, значит, что разница между двумя распределениями велика, если мала, то распределения похожи друг на друга (torch.nn.BCELoss()).
- оптимайзер
  - покажет как обновить внутренние параметры моедил, чтобы минимизровать функцию потерь
  - оптимайзеры тут torch.optim основные:
    - стохастический градиентный спуск (СГД) - на каждой итерации алгоритма из обучающей выборки каким-то (случайным) образом выбирается только один объект
    - Adam - это алгоритм оптимизации замены для стохастического градиентного спуска, может обрабатывать редкие градиенты в шумных задачах

- средняя абсолютная ошибка - на рисунке это среднее всех Difference
<img src="https://raw.githubusercontent.com/mrdbourke/pytorch-deep-learning/main/images/01-mae-loss-annotated.png" width="350" height="250">

```python
loss_fn = nn.L1Loss() # L1Loss - тоже что и средняя абсолютная ошибка


class LinearRegressionModel(nn.Module):
    def __init__(self):
        super().__init__() 
        self.weights = nn.Parameter(torch.randn(1,
                                                dtype=torch.float),
                                   requires_grad=True) 

        self.bias = nn.Parameter(torch.randn(1,
                                            dtype=torch.float),
                                requires_grad=True)

    # Forward определяет расчет в модели
    def forward(self, x: torch.Tensor) -> torch.Tensor: # <- "x" это входные данные трейн или тест
        return self.weights * x + self.bias # <- формула линейной регресии

model_0 = LinearRegressionModel()
# используем СГД, model_0 - какая нить модель, например модель линейной регресии
optimizer = torch.optim.SGD(params=model_0.parameters(), lr=0.01)
```
#### создаем обучающий луп и тестовый
- обучающий луп ходит по обучающим данным и учится на отношении features (признаков) и labels (ответов)
- тестовый луп ходит по тестовым данным и смотрит паттерны как хорошо обучилась модель

#### шаги для обучающего лупа
1. forward pass - модель проходит через обучающие данные один раз, вычисляя с помощью forward(): ```model(x_train)```
2. сalculate the loss - сравнить результаты модели с реальными данными, посмотреть какая ошибка: ```loss = loss_fn(y_pred, y_train)```
3. zero gradients - градиенты оптимизатора надо поставить на 0 (они накапливаются дефолтно), их можно пересчитать для конкретного шага обучения: ```optimizer.zero_grad()```
4. perform backpropagation on the loss - вычисляет градиент потерь относительно каждого обновляемого параметра модели (каждый параметр с require_grad=True). Это известно как **backpropagation**: ```loss.backward()```
5. update the optimizer (gradient descent) - обновите параметры с помощью require_grad=True в отношении градиентов потерь, чтобы улучшить их: ```optimizer.step()```

```python
for epoch in range(epochs)         прогнать данные через кол-во эпох:
    model.train()
    y_pred = model(x_train)
    loss = loss_fn(y_pred, y_true) посчитать лосс (как херово предиктит модель)
    optimizer.zero_grad()          надо обнулять на каждой эпохе
    loss.backward()                метод вычисления градиента, который используется при обновлении весов многослойного перцептрона )))
    optimizer.step()               обновить параметры модели согласно лосу
```

#### шаги для тестового лупа
1. forward pass	
2. calculate the loss
3. calulate evaluation metrics (опционально)

```python
torch.manual_seed(42)
epochs = 100 установим кол-во эпох
train_loss_values = []
test_loss_values = []
epoch_count = []

for epoch in range(epochs): 
    model_0.train()
    y_pred = model_0(X_train)
    loss = loss_fn(y_pred, y_train)
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()

    этап тестирования
    model_0.eval() модель переключим в процесс определения качества

    with torch.inference_mode():
        test_pred = model_0(X_test)
        test_loss = loss_fn(test_pred, y_test.type(torch.float))
        if epoch % 10 == 0: каждые 10 эпох что происходит принтануть
            epoch_count.append(epoch)
            train_loss_values.append(loss.detach().numpy())
            test_loss_values.append(test_loss.detach().numpy())
            print(f"Epoch: {epoch} | MAE Train Loss: {loss} | MAE Test Loss: {test_loss} ")
```