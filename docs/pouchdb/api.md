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
  * `options.conflicts`: 如果设置了，冲突的分支修订版将被附加在 `_conflicts` 数组。
  * `options.attachments`: 包含附件数据。
    * `options.binary`: 以 Blobs/Buffers 形式返回附件，代替 base64编码字符串形式。
  * `options.latest`: 无论请求的什么 rev，都强制获取最新的"分支"修订版，默认为 `false`

### 使用示例
```javascript
try {
  var doc = await db.get('mydoc');
} catch (err) {
  console.log(err);
}
```

### 示例响应
```json
{
  "_id": "mydoc",
  "_rev": "1-A6157A5EA545C99B00FF904EEF05FD9F",
  "title": "Rock and Roll Heart"
}
```
这个响应包含了存储在数据库中的文档，及其 `_id` 和 `_rev`.

# 删除文档
```
db.remove(doc, [options], [callback])
```
或者:
```
db.remove(docId, docRev, [options], [callback])
```
删除文档，必须至少有一个`_id`和一个`_rev`属性。发送完整的文档也可以。

请查看[过滤的复制](https://pouchdb.com/api.html#filtered-replication)以了解为什么你会想要用带有 `{_deleted: true}` 
参数的 `put()` 代替这个方法。

### 示例用法
```javascript
try {
  var doc = await db.get('mydoc');
  var response = await db.remove(doc);
} catch (err) {
  console.log(err);
}
```

### 示例响应
```json
{
  "ok": true,
  "id": "mydoc",
  "rev": "2-9AF304BE281790604D1D8A4B0F4C9ADB"
}
```
你也可以仅通过`id`和`rev`来删除一个文档:
```javascript
try {
  var doc = await db.get('mydoc');
  var response = await db.remove(doc._id, doc._rev);
} catch (err) {
  console.log(err);
}
```

你也可以通过使用带有 `{_deleted: true}` 参数的 `put()`删除一个文档:
```javascript
try {
  var doc = await db.get('mydoc');
  doc._deleted = true;
  var result = await db.put(doc);
} catch (err) {
  console.log(err);
}
```

# 创建/更新一组文档
```
db.bulkDocs(docs, [options], [callback])
```
创建，更新或删除多个文档。`docs`参数是一个文档数组。

如果你在一个给定的文档忽略了 `_id`，数据库将创建一个新文档并为你分配ID。
要更新一个文档，你必须包含 `_id`和`_rev`两个参数，该参数应与要更新的文档
的ID和修订版相匹配。最后，要删除一个文档，需要包含一个带有 `true` 值的 `_deleted`
参数。

### 使用示例
Put一些提供 `_id` 的新文档:
```javascript
try {
  var result = await db.bulkDocs([
    {title: 'Lisa Says', _id: 'doc1'},
    {title: 'Space Oddity', _id: 'doc2'}
  ]);
} catch (err) {
  console.log(err);
}
```
Post一些新文档并自动生成 `_id`:
```javascript
try {
  var result = await db.bulkDocs([
    {title: 'Lisa Says'},
    {title: 'Space Oddity'}
  ]);
} catch (err) {
  console.log(err);
}
```

### 示例响应
```json
[
  {
    "ok": true,
    "id": "doc1",
    "rev": "1-84abc2a942007bee7cf55007cba56198"
  },
  {
    "ok": true,
    "id": "doc2",
    "rev": "1-7b80fc50b6af7a905f368670429a757e"
  }
]
```
这个来自 `put()/post()` API的响应包含一个类似 `ok`/`rev`/`id` 的数组。如果有任何报错，
会单独提示如下：
```json
[
  {
    "status": 409,
    "name": "conflict",
    "message": "Document update conflict",
    "error": true
  }
]
```
返回结果的顺序与提供的"docs"数组相同。

注意`bulkDocs()`不是事务性的，你可能会返回错误/非错误的混合数组。在 CouchDB/PouchDB 里，
最小的原子单位是文档。

### 批量更新/删除:
你也可以使用 `bulkDocs()` 来一次性更新/删除很多文档:

```javascript
try {
  var result = await db.bulkDocs([
    {
      title: 'Lisa Says',
      artist: 'Velvet Underground',
      _id: 'doc1',
      _rev: '1-84abc2a942007bee7cf55007cba56198'
    },
    {
      title: 'Space Oddity',
      artist: 'David Bowie',
      _id: 'doc2',
      _rev: '1-7b80fc50b6af7a905f368670429a757e'
    }
  ]);
} catch (err) {
  console.log(err);
}
```

或者删除它们:

```javascript
try {
  var result = await db.bulkDocs([
    {
      title: 'Lisa Says',
      _deleted: true,
      _id: 'doc1',
      _rev: '1-84abc2a942007bee7cf55007cba56198'
    },
    {
      title: 'Space Oddity',
      _deleted: true,
      _id: 'doc2',
      _rev: '1-7b80fc50b6af7a905f368670429a757e'
    }
  ]);
} catch (err) {
  console.log(err);
}
```

注意: 如果你在选项对象上设定一个 `new_edits` 属性为 `false`，则可以发布其他数据库中的现有
文档，而无需为其分配新的修订版ID。通常，只有复制算法需要这样做。

# 获取一组文档
```
db.allDocs([options], [callback])
```
获取多个文档，并按`_id`索引和排序。只有 options.keys 被设置了才包括已删除的文档。

## Options
除非特殊设置否则所有选项的值都为 `false`.

  * `options.include_docs`: 在每行的`doc`字段中包含文档本身。否则，默认情况下，你只能获得
  `_id`和`_rev`属性。
  * `options.conflicts`: 在文档的 `_conflicts`字段里包含冲突信息。
  * `options.attachments`: 以base64编码淄川形式包含附件数据。
  * `options.binary`: 以 Blobs/Buffers 代替 base64 编码字符串形式返回附件。
  * `options.startkey`和`options.endkey`: 在一定范围内(包含/包含)通过 ID 获取文档.
  * `options.inclusive_end`: 包含ID等于给定`options.endkey`的文档。默认值:`true`
  * `options.limit`: 返回文档的最大数量。
  * `options.skip`: 返回时要跳过的文档数量(警告: 对IndexedDB/LevelDB是低性能的!)
  * `options.descending`: 反转输出文档的顺序。注意当 `descending: true` 时 `startkey`
  和 `endkey` 的顺序是反向的。
  * `options.key`: 仅返回ID与此字符串键相匹配的文档。
  * `options.keys`: 要在单个快照中获取的字符串键数组。
    * 此选项既不能指定 startkey 也不能指定 endkey
    * 返回的行与提供的键数组的顺序相同。
    * 删除文档的行将具有删除的修订版ID，并且在 `value` 属性里有一个额外的键 `"deleted: true"`
    * 不存在文档的行将只包含值为 `not_found` 的 `error` 属性
    * 详情信息，查阅[CouchDB 查询选项文档](https://docs.couchdb.org/en/stable/api/ddoc/views.html#db-design-design-doc-view-view-name)
    * `options.update_seq`: 包含一个`update_seq`值，该值指示视图反射的底层数据库的序列id.

注意: 对于分页，`options.limit`和`options.skip`也可用，但与CouchDB中的性能问题相同，使用
[startkey/endkey模式](https://docs.couchdb.org/en/latest/ddocs/views/pagination.html)代替。

### 示例使用:
```javascript
try {
  var result = await db.allDocs({
    include_docs: true,
    attachments: true,
  });
} catch (err) {
  console.log(err);
}
```

示例响应
```json
{
  "offset": 0,
  "total_rows": 1,
  "rows": [{
    "doc": {
      "_id": "0B3358C1-BA4B-4186-8795-9024203EB7DD",
      "_rev": "1-5782E71F1E4BF698FA3793D9D5A96393",
      "title": "Sound and Vision",
      "_attachments": {
      	"attachment/its-id": {
      	  "content_type": "image/jpg",
      	  "data": "R0lGODlhAQABAIAAAP7//wAAACH5BAAAAAAALAAAAAABAAEAAAICRAEAOw==",
      	  "digest": "md5-57e396baedfe1a034590339082b9abce"
      	}
      }
    },
   "id": "0B3358C1-BA4B-4186-8795-9024203EB7DD",
   "key": "0B3358C1-BA4B-4186-8795-9024203EB7DD",
   "value": {
    "rev": "1-5782E71F1E4BF698FA3793D9D5A96393"
   }
 }]
}
```

在响应里，你有三个东西:
  * `total_rows` 数据库中未删除文档的总数
  * 如果提供了，`偏移`这个`忽略值`，或者 CouchDB中的实际偏移
  * `rows`: 包含文档的行，如果未将 `include_docs` 设置为 true，则仅包含 `_id`/`_revs`
  * 如果你设置 `update_seq` 设置为 `true`，则还可以选择 `update_seq`

你可以使用 `startkey`/`end`来查询范围内的所有文档:
```javascript
try {
  var result = await db.allDocs({
    include_docs: true,
    attachments: true,
    startkey: 'bar',
    endkey: 'quux'
  });
} catch (err) {
  console.log(err);
}
```
这将返回所有`_id`在`'bar'`和`'quux'`之间的文档。

### 前缀搜索
你可以在 `allDocs()` 中进行前缀搜索，比如，"给我`_id`以`'foo'`开始的所有文档"，可以通过使用
特殊的高Unicode字符`'\ufff0'`:
```javascript
try {
  var result = await db.allDocs({
    include_docs: true,
    attachments: true,
    starkey: 'foo',
    endkey: 'foo\ufff0'
  });
} catch (err) {
  console.log(err);
}
```
这是因为 CouchDB/PouchDB `_id` 是按字典顺序排序的。

# 监听数据库修改

```
db.changes(options)
```
对数据库中文档所做更改的列表，按更改顺序排序。它返回一个带有 `cancel()` 方法的对象，如果不想再
监听新的更改，则调用该方法。

它是一个 event emitter，会在每个文档修改时触发 `change` 事件，当所有修改被处理后发出 `complete`
事件，当一个错误出现时一个`error`事件被触发。调用 `cancel()` 将自动取消订阅所有事件监听器。

## 选项
除非特殊设置否则所有选项都默认为 false

  * `options.live`: 将为所有未来更改触发更改事件，直到取消。
  * `options.include_docs`: 包括每次修改的相关文档。
    * `options.conflicts`: 包含冲突。
    * `options.attachments`: 包含附件。
      * `options.binary`: 作为Blobs/Buffers返回附件数据，代替base64编码字符串。
  * `options.descending`: 反序输出文档。
  * `options.since`: 在给定的序列号之后立即开始更改结果。你只需要新的更改(当`live`为`true`时)
  也可以传递`'now'`
  * `options.limit`: 限制结果的数量到这个数字。
  * `options.timeout`: 请求超时(毫秒)，使用 `false` 来关闭。
  * `options.heartbeat`: 仅针对 http 适配器，服务器发出心跳以保持长连接打开的以毫秒为单位的
  时长。默认为10000(10秒)，默认使用false来关闭。

### 过滤选项:
  * `options.filter`: 从设计文档中引入一个过滤函数以有选择的获取更新。要是用视图函数，在这里
  传递`_view`并在 `options.view` 里提供一个到该视图函数的引用。查看过滤修改以了解更多。
  * `options.doc_ids`: 仅显示这些id(字符串数组)的文档的更改。
  * `options.query_params`: 一个包含传递给筛选函数的属性的对象，如 `{foo:"bar"}`, 这里
  `"bar"`将在过滤函数里作为 `params.query.foo` 可用。要访问 `params`，定义你的过滤器函数，
  如 `function(doc, params) {/* ... */}`
  * `options.view`: 指定一个视图函数(如 `design_doc_name/view_name`或`view_name`作为
  `'view_name/view_name'的缩写`)作为一个过滤器。如果映射函数至少为文档发出一条记录，则被视为
  视图过滤器"通过"的文档。*注意*:`options.filter`必须设置到`_view`，这个选项才能工作。
  * `options.selector`: 使用query/pouchdb-find选择器进行筛选。注意：CouchDB 1.x不支持
  选择器。不能与过滤器选项结合使用。

### 高级选项:
  * `options.return_docs`: 默认为`true`，除非`options.live = true`默认才为`false`.
  传递 `false` 以防止修改提要将所有文档都保存在内存中，换句话说，总有一个空结果数组，`change`
  事件是唯一获取此事件的方法。对于较大的修改集非常有用，否则会耗尽内存。
  * `options.batch_size`: 仅对 http 数据库可用，这里配置一次获取多少更改。增加这个值可以
  减少请求次数。默认值为25.
  * `options.style`: 指定在修改数组里返回多少修订版。默认值是 `'main_only'`，只返回当前
  `winning`修订版；`'all_docs'`将返回所有分支修订版(包括冲突和删除的以前的冲突)。很可能你不
  需要这个，除非你正在写一个复制程序。
  * `options.seq_interval`: 仅适用于http数据库。指定每N次修改后才生成seq信息。较大的值可以
  提高 CouchDB 2.0及更高版本的吞吐量。注意始终填写 `last_seq`。

## 使用示例:
```javascript
var changes = db.changes({
  since: 'now',
  live: true,
  include_docs: true
}).on('change', fucntion(change) {
  // handle change
}).on('complete', fucntion(info) {
  // changes() was canceled
}).on('error', function(err) {
  console.log(err);
});

change.cancel();
```

```json
{
  "id":"somestuff",
  "seq":21,
  "changes":[{
    "rev":"1-8e6e4c0beac3ec54b27d1df75c7183a8"
  }],
  "doc":{
    "title":"Ch-Ch-Ch-Ch-Changes",
    "_id":"someDocId",
    "_rev":"1-8e6e4c0beac3ec54b27d1df75c7183a8"
  }
}
```

## 修改事件
  * `change`(`info`) - 当一个修改发生时触发此事件。`info`将包含这次修改的明细，比如它是否
  被删除了和最新的 `_rev` 是什么。
  * `complete`(`info`) - 当所有更改被读取后触发这个事件。在实时更改后，只有取消更改才会触发
  此事件。`info.results`将包含更改列表。请参见下面的示例。
  * `error`(`err`) - 由于不可恢复的故障而停止更改源时，将触发此事件。

## 示例响应

在 `'change'` 监听器里的示例响应(使用 `{include_docs: true}`):
```javascript
{ 
  id: 'doc1',
  changes: [ { rev: '1-9152679630cc461b9477792d93b83eae' } ],
  doc: {
    _id: 'doc1',
    _rev: '1-9152679630cc461b9477792d93b83eae'
  },
  seq: 1
}
```
当一个文档被删除后在 `'change'` 监听器里示例响应:
```javascript
{ 
  id: 'doc2',
  changes: [ { rev: '2-9b50a4b63008378e8d0718a9ad05c7af' } ],
  doc: { _id: 'doc2',
    _rev: '2-9b50a4b63008378e8d0718a9ad05c7af',
    _deleted: true
  },
  deleted: true,
  seq: 3
}
```
在`'complete'`监听器里的示例响应:
```json
{
  "results": [
    {
      "id": "doc1",
      "changes": [ { "rev": "1-9152679630cc461b9477792d93b83eae" } ],
      "doc": {
        "_id": "doc1",
        "_rev": "1-9152679630cc461b9477792d93b83eae"
      },
      "seq": 1
    },
    {
      "id": "doc2",
      "changes": [ { "rev": "2-9b50a4b63008378e8d0718a9ad05c7af" } ],
      "doc": {
        "_id": "doc2",
        "_rev": "2-9b50a4b63008378e8d0718a9ad05c7af",
        "_deleted": true
      },
      "deleted": true,
      "seq": 3
    }
  ],
  "last_seq": 3
}
```
`seq`和`last_seq`对应整个数据库的总序列号，它是使用`since`时传入的序列号(除了特殊的`'now'`).
它是更改提要的主键，也被复制算法用作检查点。

## 单发

如果你没设定 `{live: true}`, 你也可以在标准的callback/promise风格中使用`changes()`, 其
被视为一个单次请求，其返回一个更改列表(即`'complete'`事件发出的内容):
```javascript
// callbacks style
db.changes({
  limit: 10,
  since: 0
}, function (err, response) {
  if (err) { return console.log(err); }
  // handle result
});

// Promises
db.changes({
  limit: 10,
  since: 0
}).then(function (result) {
  // handle result
}).catch(function (err) {
  console.log(err);
});

// Async functions
try {
  var result = await db.changes({
    limit: 10,
    since: 0
  });
} catch (err) {
  console.log(err);
}
```

## 响应示例:
```json
{
  "results": [{
    "id": "0B3358C1-BA4B-4186-8795-9024203EB7DD",
    "seq": 1,
    "changes": [{
      "rev": "1-5782E71F1E4BF698FA3793D9D5A96393"
    }]
  }, {
    "id": "mydoc",
    "seq": 2,
    "changes": [{
      "rev": "1-A6157A5EA545C99B00FF904EEF05FD9F"
    }]
  }, {
    "id": "otherdoc",
    "seq": 3,
    "changes": [{
      "rev": "1-3753476B70A49EA4D8C9039E7B04254C"
    }]
  }, {
    "id": "828124B9-3973-4AF3-9DFD-A94CE4544005",
    "seq": 4,
    "changes": [{
      "rev": "1-A8BC08745E62E58830CA066D99E5F457"
    }]
  }],
  "last_seq": 4
}
```

当 `live` 是 `false` 时，返回的对象也是一个事件发射器和一个promise，当结果准备就绪时将触发
`'complete'`事件。

注意，此`'complete'`事件只有当你不进行实时更改时触发。

### 过滤更改
与replicate()一样，可以使用以下方法进行筛选:
  * 一个特殊的 `filter` 函数
  * `doc_ids`的数组
  * 设计文档中的`filter`函数
  * 设计文档中的`filter`函数，带有`query_params`
  * 设计文档中的`view`函数

如果在一个远程 CouchDB 上运行 `changes()`, 那么第一个方法将在客户端运行，而最后四个方法将在
服务器端进行过滤。因此应首选最后四个，尤其是在数据库很大的情况下，因为你希望通过网络发送尽可能少
的文档。

如果你在本地 PouchDB 上运行 `change()`，那么显然这五个方法都将在客户端运行。使用这五种方法中
的任何一种都不会带来性能上的好处，因此您可以在自己的 `on('change')`处理器里对你自己进行筛选。
这些方法在 PouchDB 里的实现纯粹是为了与 CouchDB 保持一致。

命名函数可以用 `designdoc_id/function_name` 指定或者(如果design doc id和function name相等)
用`'fname'`作为`'fname/fname'`的简称。

## Filtering examples
In these examples, we’ll work with some mammals. Let’s imagine our docs are: