---
title: RESTful API 分页问题
last_modified_at: 2019-12-19T23:20:02+08:00
classes: wide
categories:
  - 2019-11
tags:
  - django
  - rest-api
  - web-dev
---


传统的 REST API 分页方式`?page=1&size=10`主要应用于静态的或者不经常改变的数据。但面对动态的或者实时数据，就会造成数据重复显示或丢失，尤其在移动端上表现更为明显。因为几乎每个移动应用都会有上拉加载更多数据的交互。

网上有很多相关的讨论文章，比如，[这里](https://www.sitepoint.com/paginating-real-time-data-cursor-based-pagination/)所举的例子，20条数据逆序排列，第一页请求第 20 到第 11 条数据。

![page 1](/assets/images/2019/11/page1.png)

如果这时又有5条新增数据（21到25），那么使用 `?page=2` 请求第二页数据就会返回第 15 到第 6 条数据，此时就会有5条重复数据。

![page 2](/assets/images/2019/11/page2.png)

同理，如果恰巧删除了5条数据（20到16），那么使用 `?page=2` 请求第二页数据就会返回第 5 到第 1 条数据，此时就会有5条数据“丢失”！

## 解决方案

这几年业界普遍使用游标分页。比如，twitter 搜索 api 请求最新的 10 条包含 php 的帖子，增加了 `since_id` 和 `max_id`:

```
https://api.twitter.com/1.1/search/tweets.json?q=php&since_id=24012619984051000&max_id=250126199840518145&result_type=recent&count=10
```

返回结果包含 metadata 信息：

```
"search_metadata": {
  "max_id": 250126199840518145,
  "since_id": 24012619984051000,
  "refresh_url": "?since_id=250126199840518145&q=php&result_type=recent&include_entities=1",

  "next_results": "?max_id=249279667666817023&q=php&count=10&include_entities=1&result_type=recent",

  "count": 10,
  "completed_in": 0.035,
  "since_id_str": "24012619984051000",
  "query": "php",
  "max_id_str": "250126199840518145"
}
```

其中，包含了下一页的 url，客户端请求这个 url 就可以准确获得下一页的10条数据。 服务端根据 `max_id` 从数据库里过滤10条数据。

## 游标分页 (cursor pagination)

Django REST framework 提供的[游标分页](https://www.django-rest-framework.org/api-guide/pagination/#cursorpagination)，把游标当前位置以及 `offset` 信息通过 base64 方式编码到 url 参数里。

```python
    def encode_cursor(self, cursor):
        """
        Given a Cursor instance, return an url with encoded cursor.
        """
        tokens = {}
        if cursor.offset != 0:
            tokens['o'] = str(cursor.offset)
        if cursor.reverse:
            tokens['r'] = '1'
        if cursor.position is not None:
            tokens['p'] = cursor.position

        querystring = parse.urlencode(tokens, doseq=True)
        encoded = b64encode(querystring.encode('ascii')).decode('ascii')
        return replace_query_param(self.base_url, self.cursor_query_param, encoded)
```

使用时需要注意的地方：

- 分页用到的字段必须是不可更改的，而且有数据库索引。例如创建时间。

- 必须唯一或近似唯一。例如精确到微秒级别的时间戳。


当然，如果数据量大到微秒级别的时间戳也会重复时，[这里有介绍](https://blog.vermorel.com/journal/2015/5/8/nearly-all-web-apis-get-paging-wrong.html)使用 “continuation token” 模式。不过，实现起来会更加复杂，等实际需要用到的时候再研究研究。

## 参考

- [https://stackoverflow.com/questions/35462966/how-to-ensure-data-consistency-in-rest-api-pagination](https://stackoverflow.com/questions/35462966/how-to-ensure-data-consistency-in-rest-api-pagination)

- [https://www.moesif.com/blog/technical/api-design/REST-API-Design-Filtering-Sorting-and-Pagination/](https://www.moesif.com/blog/technical/api-design/REST-API-Design-Filtering-Sorting-and-Pagination/)

- [https://cra.mr/2011/03/08/building-cursors-for-the-disqus-api](https://cra.mr/2011/03/08/building-cursors-for-the-disqus-api)
