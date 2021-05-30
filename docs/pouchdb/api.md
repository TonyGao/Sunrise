# API 概述
PouchDB 有一个异步API，支持回调、promise 和 async函数。对于初学者，我们建议使用 promise，
尽管你可以自由地使用任何你喜欢的格式。如果你不确定，请查看我们的异步代码指南。

大多数 API 公开为:
```javascript
db.doSomething(args..., [options], [callback])
```
...里边 `options`和`callback`是选项。

### Callbacks
Callback 使用标准的 Node.js 习惯用法：
```javascript
function(error, result) { /* ... */ }
```
...里边如果没有错误 error 将不会定义。

### Promise
如果你没设定一个 `callback`，那么 API 会返回一个 promise. 在支持的浏览器或Node.js，
原生promise将被使用，根据需要返回到一个最小库 [lie](https://github.com/calvinmetcalf/lie).

>你使用Ionic/AngularJS吗?你可以包裹 PouchDB promise 在 $q.when() 里。这将通知 AngularJS
> 当 PouchDB promise 处理时更新 UI.

要是用自定义Promise与PouchDB一起使用，你必须在加载 PouchDB 之前重定义全局的 Promise 对象:

```html
<script>window.Promise = MyCustomPromiseLibrary;</script>
<script src="path/to/pouchdb.js"></script>
```

### Async 函数
如果你使用类似 Babel 的转换器，你可以开启 async 函数，其实一个实验性的 API 临时发布会于ES7(ES2016)
中。其允许你当尝试类似 PouchDB 这种以 promise 为基础的API时使用 async/await 关键字。

>怎样配置Babel? 要是用 async 函数，你将需要 syntax-async-function 插件，以及 transform-regerator
>插件或Kneden>(在拟写本文时它是实验性的)。有关完整的工作示例，请参见[async-functions-with-regenerator](https://github.com/>nolanlawson/async-functions-with-regenerator)


注意在 API 文档中的 async/await 的示例都假设你的代码已经在一个 async 函数内了。如下实例：
```javascript
async function myFunction() {
	// your code goes in here
}
```

任何没在一个 async 函数里的 `await` 都是语法错误。关于 async/await 的更多信息，请阅读我的产业博客文章。

# 创建一个数据库

```javascript
new PouchDB([name], [options])
```

这个方法创建一个数据库或打开一个已有的数据库。如果你使用一个类似 `'http://domain.com/dbname'` URL，
然后 PouchDB 将作为一个 CouchDB 的在线实例的客户端工作。否则它将创建一个本地数据库，无论后端怎样。

### 选项
  * name: 你可以忽略 name 参数，并以通过 `options` 来代替。请注意 name 是必须的。

*本地数据库选项*

  * `auto_compaction`: 这将开启自动压缩，这意味着每次更新数据库后都会调用 `compact()`。默认为 false
  * `adapter`: `idb`, `leveldb`, or `http` 的其中之一
  * `revs_limit`: 指定我们跟踪的版本数量(不是副本)。指定一个低值意味着Pouch可能无法确定通过复制接受的
  新修订是否与它当前拥有的任何可能导致冲突的修订相关。默认为`1000`
  * `deterministic_recs`: 使用 md5 哈希为文档来创建一个确定性修订号。设定为 false 意味着修订号将是
  一个随机的 UUID。默认为 true

### 远程数据库的选项

  * `fetch(url, opts)`: 拦截或覆盖 HTTP 请求，你可以添加或修改任何设计http请求的头信息或选项，然后返回
  一个 fetch Promise
  * `auth.username` + `auth.password`: 你可以使用以下形式指定 HTTP 验证参数来使用数据库，
  通过`http://user:pass@host/name` 或通过 `auth.username` + `auth.password` 选项。
  * `skip_setup`: 初始化 PouchDB 检查是否数据库存在，并且如果它不存在就试着创建它。设置为 true 则跳过
  这个设置。

  ### 注意

    1. 在 IndexedDb 里PoutchDB将使用 `_pouch_` 作为数据库前缀。不要手动创建同前缀的数据库。
	2. 当作为 Node 的客户端时，任何其他选项都将传递给请求。
	3. 当使用`leveldb`适配器(Node上的默认值)，任何其他选项都将传递给 `levelup`

### 使用实例

```javascript
var db = new PouchDB('dbname');
// or
var db = new PouchDB('http://localhost:5984/dbname');
```

创建一个内存级 Pouch(必须首先安装 `pouchdb-adapter-memory`):
```javascript
var db = new PouchDB('dbname', {adapter: 'memory'});
```
以一个特殊的 fetch 选项创建一个远程 PouchDB:
```javascript
var db = new PouchDB('http://example.com/dbname', {
	fetch: function(url, opts) {
		ops.headers.set('X-Some-Special-Header', 'foo');
		return PouchDB.fetch(url, opts);
	}
});
```
更多信息，请查看[适配器](https://pouchdb.com/adapters.html)

# 删除一个数据库
```
db.destroy([options], [callback])
```
删除此数据库。请注意，这对其他复制的数据库没有影响。

### 使用示例
```javascript
try {
	await db.destroy();
} catch (err) {
	console.log(err);
}
```

示例响应
```javascript
{
	"ok": true
}
```

# 创建/更新一个文档

## 使用 db.put()
```
db.put(doc, [options], [callback])
```

创建一个新文档或更新一个已有文档。如果此文档已存在，你必须设定它的修订号 `_rev`，否则会出现冲突。

如果你想要更新一个已有文档，甚至当它发生冲突，你必须设定基础修订号 `_rev` 并使用 `force=true` 选项，然后会创建
一个新的冲突修订号。

`doc` 必须是一个 "纯的 JSON 对象"，如，一个 name/value 对集合。如果你尝试存储非JSON数据(比如 Date 对象)
你可以查看 [不一致的结果](https://pouchdb.com/errors.html#could_not_be_cloned)

### 使用示例：
以`_id` 为 `'mydoc'`创建一个新文档：

```javascript
try {
  var response = await db.put({
    _id: 'mydoc',
    title: 'Heroes'
  });
} catch (err) {
  console.log(err);
}
```

你可以使用 `_rev` 更新一个已有文档:
```javascript
try {
  var doc = await db.get('mydoc')
  var response = await db.put({
    _id: 'mydoc',
    _rev: doc._rev,
    title: "Let's Dance"
  });
} catch(err) {
  console.log(err);
}
```

### 示例响应:
```json
{
  "ok": true,
  "id": "mydoc",
  "rev": "1-A6157A5EA545C99B00FF904EEF05FD9F"
}
```

此响应包含这个文档的 `id`，新的 `rev`, 并用一个 `ok` 安抚你一切都很好。

## 使用 db.post()
```
db.post(doc, [options], [callback])
```
创建一个文档并让 PouchDB 为它自动生成一个 `_id`

### 使用示例

```javascript
try {
  var response = await db.post({
    title: 'Ziggy Stardust'
  });
} catch (err) {
  console.log(err);
}
```

示例响应:
```json
{
  "ok": true,
  "id": "8A2C3761-FFD5-4770-9B8C-38C33CED300A",
  "rev": "1-d3a8e0e5aa7c8fff0c376dac2d8a4007"
}
```

*Put vs post*: 基本的经验法则是: `put()`有 `_id`的新文档，`post()`没有 `_id`的新文档。

你还是应该更青睐 `put()`而不是`post()`，因为当你是用`post()`，你就失去了一个使用 `allDocs()`通过 `_id`
进行文档排序的机会(因为你的 _id 是随机的). 要查看更多信息，请阅读[PouchDB 高级技巧](https://pouchdb.com/2014/06/17/12-pro-tips-for-better-code-with-pouchdb.html)

# 读取一个文档

```
db.get(docId, [options], [callback])
```
检索由 `docId` 指定的文档。

## 选项
除非针对设置，否则所有选项默认值都为 `false`。

  * `options.rev`: 获取一个文档的特定修订版。默认为winning(获胜)修订版(查阅 [CouchDB指南](http://guide.couchdb.org/draft/conflicts.html))
  * `options.revs`: 包括文档的修订历史记录。
  * `options.revs_info`: 包括一个文档的修订列表，和它们的可用性。
  * `options.open_revs`: 如果 `open_revs="all"` 则获取所有分支修订版或者获取所有设置在 `open_revs` 数组中的分支修订版。
  分支也将以这个设置的输入数组的顺序进行排序。
  * `options.conflicts`: 

