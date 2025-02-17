---
title: 'Svable — простая генерация SVG на сервере'
date: 2013-12-13
author: ilya-zayats
editors:
  - vadim-makeev
  - olga-aleksashenko
layout: article.njk
tags:
  - article
  - js
  - svg
---

_Мы начинаем серию статей, в которых вы можете рассказать о своём сервисе или разработке, которые могут быть полезны для фронтенд-разработчиков. Есть о чём рассказать? Пишите нам на [wst@web-standards.ru](mailto:wst@web-standards.ru). Редакция._

Рассказывать про актуальность векторной графики уже нет никакого смысла — количеством различных размеров, разрешений (а теперь еще и плотностей) экранов веб-разработчика не удивишь. А вот вопрос подготовки и создания изображений для всего этого разнообразия набирает актуальность с каждым днем. В этой статье я хочу рассказать не о том, как научить дизайнера использовать Illustrator, а о том, как облегчить жизнь разработчика при работе с SVG.

В последнее время SVG начинают использовать не только в качестве замены растра для иконок и логотипов начинает, но и для создания сложной графики и динамических диаграмм. За это нужно сказать спасибо множеству отличных JavaScript-библиотек, которые позволяют реализовывать все фантазии дизайнеров быстро и легко: Raphaël, Snap.svg, SVG.js, а также тех, что визуализируют разнообразные данные в любом вообразимом виде: D3, Highcharts, GRaphaël.

Но я говорил об облегчении жизни разработчиков: в чем она нелегка, если хороших инструментов так много? Проблемы начинаются, когда перед программистами ставится одна из следующих задач:

- отрисовать графики на сервере, чтобы вставить в email-рассылку;
- закэшировать результат, потому что каждая отрисовка очередного сложного изображения «подвешивает» браузер на слабой машине;
- дать пользователю возможность скачать файл в PNG или PDF.

Очевидно, основное преимущество всех этих библиотек становится их самым большим минусом — они работают только в браузере.

Решения этой проблемы уж больно прямолинейные — перетащить браузер на сервер. Тут и решения «headless», например, у [highcharts](http://www.highcharts.com/component/content/article/2-news/52-serverside-generated-charts) или [freckle](http://mir.aculo.us/2013/04/30/embed-canvas-and-svg-charts-in-emails/), или попытки перенести [DOM в Node.js для Raphaël](https://github.com/dodo/node-raphael).

Это работает, но инфраструктура для столь небольших потребностей получается монструозной, сложной и часто медленной.

Почему же тогда просто не использовать сторонние серверные библиотеки? Ни один программист в мире не захочет поддерживать две реализации одного и того же на двух разных языках. Это ведет к ошибкам, дополнительным сложностям и высокой итоговой стоимости поддержки и разработки новой функциональности.

Мы все это понимаем, поэтому и создали [Svable](http://svable.com "Svable"). Это сервис, который позволит вам перенести всю вашу существующую генерацию SVG из браузера на сервер или запускать один и тот же код как на клиенте, так и на сервере.

Как это реализуется? Система состоит из двух частей: самой платформы и адаптеров. Платформа — это API, который принимает на вход специально сформированный JSON с высокоуровневыми командами, вроде: `rect`, `circle`, `getBBox` и т.д. В ответ вы получаете результирующий SVG, PDF или PNG. Мы конвертируем результат, если вам это нужно.

Возьмем для примера [вот этот SVG](http://codepen.io/anon/pen/jiHkq). Как видим, это круг с полупрозрачным `stroke` и белым квадратом, который рисуется ровно по центру этого круга. И все это на фоне белого прямоугольника с закругленными краями. Допустим, нам позарез нужно отрисовать подобное на сервере. Для этого потребуется сформировать POST-запрос на наш сервис со следующим JSON:

```json
{
    'paper': {
    'attrs': [ { 'width': '640' }, { 'height': '480' }],
    'access_key': 'my_key',
    'format': 'svg',
    'children': [
        {
        'type': 'rect',
        'attrs': [
            { 'x': '0' },
            { 'y': '0' },
            { 'width': '640' },
            { 'height': '480' },
            { 'rx': '10' },
            { 'fill': '#fff' }
        ]},
        {
        'type': 'circle',
        'svable_id': 'main_circle',
        'attrs': [
            { 'cx': '320' },
            { 'cy': '240' },
            { 'r': '60' },
            { 'fill': '#223fa3' },
            { 'stroke': '#000000' },
            { 'stroke-width': '80' },
            { 'stroke-opacity': '0.5' }
        ]},
        {
        'type': 'rect',
        'attrs': [
            { 'x': 'main_circle.cx - 10' },
            { 'y': 'main_circle.cy - 10' },
            { 'fill': '#fff' },
            { 'width': '20' },
            { 'height': '20' }
        ]}
    ]}
}
```

Разберем его по частям: в корневом объекте `paper` вы описываете размеры вашего полотна, `viewBox`, а также указываете ваш персональный ключ доступа и растровый формат (по умолчанию в ответ вам придет SVG):

```json
'paper': {
    'attrs': [{ 'width': '640' }, { 'height': '480' }],
    'access_key': 'my_key',
    'format': 'svg',
```

В массиве `children` указываются все объекты, которые попадут на ваш холст. Первым из них идет фоновый прямоугольник. Тут все достаточно просто и почти один в один повторяет формат самого SVG-документа:

```json
'type': 'rect',
'attrs': [
    { 'x': '0' },
    { 'y': '0' },
    { 'width': '640' },
    { 'height': '480' },
    { 'rx': '10' },
    { 'fill': '#fff' }
]
```

Далее опишем круг. Тут все тоже довольно банально, но есть одно отличие — атрибут `svable_id`, который позволит вам сослаться конкретно на этот объект в тот момент, когда вам понадобятся любые его параметры:

```json
'type': 'circle',
'svable_id': 'main_circle',
'attrs': [
    { 'cx': '320' },
    { 'cy': '240' },
    { 'r': '60' },
    { 'fill': '#223fa3' },
    { 'stroke': '#000000' },
    { 'stroke-width': '80' },
    { 'stroke-opacity': '0.5' }
]
```

Затем опишем последний квадрат. Напомним, что он должен позиционироваться относительно центра круга. Тут вам и пригодится то, что вы указали в `svable_id`:

```json
'type': 'rect',
'attrs': [
    { 'x': 'main_circle.cx - 10' },
    { 'y': 'main_circle.cy - 10' },
    { 'fill': '#fff' },
    { 'width': '20' },
    { 'height': '20' }
]
```

Однако обычно никто не рисует SVG на сервере с нуля. Поэтому мы создали адаптеры под популярные JavaScript-библиотеки: уже готов Raphaël, завершаем работу над Snap.svg и D3.

Адаптеры как раз занимаются тем, что преобразовывают код, написанный для браузерных библиотек в JSON, который понимает наша платформа. В итоге вы легко можете запускать свой код там, где вам сейчас это выгодно, лишь вызовите адаптер в нужный момент.

Возьмем уже знакомый нам SVG и [отрисуем с помощью Raphaël](http://codepen.io/anon/pen/iJext):

```js
var paper = Raphael(0, 0, 640, 480);
    paper
        .rect(0, 0, 640, 480, 10)
        .attr({
            fill: '#fff',
            stroke: 'none'
});

var circle = paper
    .circle(320, 240, 60)
    .attr({
        fill: '#223fa3',
        stroke: '#000',
        'stroke-width': 80,
        'stroke-opacity': 0.5
});

paper
    .rect(circle.attr('cx') - 10, circle.attr('cy') - 10, 20, 20)
    .attr({
        fill: '#fff',
        stroke: 'none'
});
```

А теперь перенесем его на сервер и [отрисуем с помощью Node.js](http://codepen.io/anon/pen/woJgA):

```js
var Svable = require('svable');
var paper = Svable(0, 0, 640, 480, 'raphael');

paper
    .rect(0, 0, 640, 480, 10)
    .attr({
    fill: '#fff',
    stroke: 'none'
});

var circle = paper
    .circle(320, 240, 60)
    .attr({
        fill: '#223fa3',
        stroke: '#000',
        'stroke-width': 80,
        'stroke-opacity': 0.5
});

paper
    .rect(circle.attr('cx') - 10, circle.attr('cy') - 10, 20, 20)
    .attr({
        fill: '#fff',
        stroke: 'none'
});

console.log(paper.burnSync());
```

Как видите, вся разница лишь в первоначальном вызове объекта и итоговом получении результата.

В данном случае вызов `paper.burnSync()` синхронно вернет нам необходимый XML, с которым вы уже можете поступить как захотите. Любители асинхронности могут воспользоваться методом `burn` — он возвращает промис.

Таким образом, вы получаете возможность вставить итоговый файл в почтовую рассылку, отдать пользователю на скачивание, сохранить на сервере, чтобы в следующий раз сэкономить как время пользователя, так и деньги на генерации, и все это без дублирования кода и страданий программистов.

Мы сейчас находимся на финальной стадии разработки, и нам нужны первые клиенты со сложными задачами для пробных интеграций. Мы не только решим их проблемы, но и дадим выгодные условия на время бета-теста. Напишите нам, обсудим конкретно ваш случай: [svable.com](http://svable.com/).
