---
layout: post
title: Баг сериализации тела Http запроса
tags: c# .Net aspnetcore
comments: true
---

На этой неделе на работе я столкнулся с одним неприятным багом. При тестировании взаимодействия двух микросервисов через http, на принимающей стороне всегда вместо ожидаемого объекта приходил null. Однако, если отправлять запрос через Postman, все работало ожидаемо. Пришлось потратить пару часов, чтобы разобраться в чем же дело.

<!--more-->
Казалось бы, будничная ситуация - есть два сервиса. Один, написанный на dotnetcore, создает HttpRequestMessage, кладет в Content объект и шлет его POSTом в контроллер второго сервиса, написанный на dotnet framework, который сериализует тело запроса в объект.

Код первого сервиса:
```C#
    var request = new HttpRequestMessage(HttpMethod.POST, uri);
    request.Content = new ObjectContent<MyData>(data, new JsonMediaTypeFormatter());
    var response = await HttpClient.SendAsync(request, cancellationToken);
```

Код второго сервиса:
```C#
    public class MyControllerController : ApiController
    {
        [HttpPost]
        [Route("do_somework")]
        public async Task<IHttpActionResult> DoSomething([FromBody] MyData data) // вот здесь приходит null
        {
            await Somework(data);
            return Ok();
        }
    }
```

Сравнивая запросы посылаемые сервисом и Postmanом, я обнаружил, что в случае запроса из сервиса в заголовке запроса Content-Length всегда был 0, а из Postman - нужной длины. Дело оказалось в ObjectContent. Он сериализовал объект в Json,
но почему-то не задавал длину контента для запроса. Кстати, так же я узнал, что нельзя вместо Content-Length указать просто
длину строки, если в Json есть русские буквы :) - русские буквы будут 2 байтовыми, а английские - однобайтовыми.

Ну и как результат нашел способ найти это сделать правильно: 

```C#
    var request = new HttpRequestMessage(HttpMethod.POST, uri);
    request.Content = new StringContent(JsonConvert.SerializeObject(data), Encoding.UTF8, "application/json");
    var response = await HttpClient.SendAsync(request, cancellationToken);
```
Баг побежден, работаем дальше :) 