---
layout:     post
title:      "Python requests库的stream属性"
date:       2019-06-25 15:55:42
author:     "Jonliu"
header-img: "img/post-bg-2018.jpg"
tags:
    - Python
---

## Requsts stream属性介绍
官方文档描述如下：
>iter_content(chunk_size=1, decode_unicode=False)[source]

Iterates over the response data. When stream=True is set on the request, this avoids reading the content at once into memory for large responses. The chunk size is the number of bytes it should read into memory. This is not necessarily the length of each item returned as decoding can take place.

chunk_size must be of type int or None. A value of None will function differently depending on the value of stream. stream=True will read data as it arrives in whatever size the chunks are received. If stream=False, data is returned as a single chunk.

If decode_unicode is True, content will be decoded using the best available encoding based on the response.

> iter_lines(chunk_size=512, decode_unicode=False, delimiter=None)[source]

Iterates over the response data, one line at a time. When stream=True is set on the request, this avoids reading the content at once into memory for large responses.

requests中会默认stream=False，仅开启stream模式的时候需要才调用iter_content和iter_lines。

## 使用stream属性的注意事项

- 1.设置stream=True的时，例如: `r = requests.get(url, stream=True)`，在调用r.iter_content或r.iter_lines之前，仅仅打开了IO连接，并获取了response headers，此时你可以对资源的Content-Type和Content-lenth进行判断。

- 2.在使用response迭代完所有body内容后，并不会关闭IO连接，需要显示地调用close()。但是更优雅的方式是使用with，例如:`with requests.get(url, stream=True) as r:`。

## 使用案例

### 一、使用案例

1.requests的timeout“失效”：
> requests的timeout在某些情况下会失效。首先它提供的timeout实际上是connect_timeout和read_timeout之和，前者是创建IO连接的超时时间，后者是连接后到读取第一个bytes之间的超时时间。而一旦读取到bytes，requests定义的timeout参数就“失效”了，这个连接很可能会花费超长的时间，例如：读取一个大小为10G的文件。所以，这个时候建议打开stream属性，用分片的方式去拉取数据。

2.需要下载进度百分比：
> 通过获取Content-length和每次调用iter_contet，能够得到当前下载的bytes百分比，即下载进度。

3.需要断点续传:
> 同上，能够知道上次下载的位置(已下载的bytes数)

4.判断整个requests请求是否超时（包括开始read bytes之后的时间）

5.判断response返回的body体是否超过限制

### 二、代码示例

1.下载进度百分比

```python
def get_download_percent(r: requests.Response) -> float:
    bytes_container = BytesIO()
    bytes_total = r.headers['Content-length']
    for chunk in r.iter_content(chunk_size=8192):
        bytes_container.write(chunk)
        bytes_count = len(bytes_container.getvalue())
        yield bytes_count / bytes_total
```

2.判断requests请求返回的类型和内容大小是否合法代码示例:

```python
def is_request_time_and_length_valid(r: requests.Response) -> bytes:
    bytes_container = BytesIO()
    time_begin = time.time()

    for chunk in r.iter_content(chunk_size=8192):
        if time.time() - time_begin >= REQUEST_MAX_TIME_COST:
            raise RequestTooLongTimeException()
        bytes_container.write(chunk)

    resp_content = bytes_container.getvalue()
    if len(resp_content) >= REQUEST_MAX_CONTENT_BYTES:
        raise ResponseTooManyBytesException()

    return resp_content
```

## 参考

[1][Requests文档](https://2.python-requests.org/en/master/user/advanced/)