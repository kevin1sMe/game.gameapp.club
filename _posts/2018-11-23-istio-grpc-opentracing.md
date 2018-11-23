---
layout: post
title:  "Istio中对grpc全跟踪"
author: kevin
categories: tools
tags: grpc envoy istio tracing opentracing
---
* content
{:toc}

>项目内（预研）在使用`Istio`的默认网关`envoy`作为service mesh的最要组件。其本身可以配置zipkin或jaeger等opentracing的实现作为业务调用追踪(tracing)监控方案。但如果只依靠于envoy本身的tracing，则会在靠近业务侧中断，使前后请求响应不能串联，所以业务本身需要负责此串联。花了些时间看了一下，使用Python基于opentracing实现了一个串联grpc的方案。

#### 前置资料
##### 背景 
* 业界已经有几种Tracing方案，如ebay、点评的ACT方案。
* google的论文Dapper出来后，形成了一个通用的tracing标准，即`OpenTracing`。

##### OpenTracing基础框架
* OpenTracing视角看请求
![Alt text](./assets/201811/istio-grpc-tracing-01.png)
主要需要了解Tracer, Span, SpanContext, Carrier, Baggage等概念，具体自行查阅[资料]((https://opentracing.io/specification/))。

* [zipkin](https://zipkin.io/)是一个opentracing的实现，稍了解下其基本框架。
![Alt text](./assets/201811/istio-grpc-tracing-zipkin.png)

* [jaeger](https://www.jaegertracing.io)是基于zipkin的实现，同样实现了opentracing。
![Alt text](./assets/201811/istio-grpc-tracing-jaeger.png)

#### 业务目标
1. 打通envoy及业务的tracing
2. 对grpc请求进行tracing

#### 实现
##### envoy侧
研究envoy的tracing机制，可见其官方[文章](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/tracing#arch-overview-tracing) 。支持的tracing已经挺丰富了。
>envoy.lightstep
envoy.zipkin
envoy.dynamic.ot
envoy.tracers.datadog

我们使用jaeger来做追踪，默认配置如下：
```
tracing:
  http:
    name: envoy.zipkin
    config:
      collector_cluster: jaeger
      collector_endpoint: "/api/v1/spans"
```
对于jaeger(zipkin)的Tracing方式，envoy中大概是这样的过程：
>When using the Zipkin tracer, Envoy relies on the service to propagate the B3 HTTP headers ( x-b3-traceid, x-b3-spanid, x-b3-parentspanid, x-b3-sampled, and x-b3-flags). The x-b3-sampled header can also be supplied by an external client to either enable or disable tracing for a particular request. In addition, the single b3 header propagation format is supported, which is a more compressed format.

即在头部有如上的一些消息，当然也支持b3的压缩格式: xx:yy:zz:aa。以实际中收包的metadata中打出来的如下：
```
act_server_1  | x-b3-sampled 1
act_server_1  | x-request-id 91fac7d8-ad0f-9a04-84f6-88a8192d08fe
act_server_1  | x-b3-parentspanid 1a6d063b2e392c71
act_server_1  | x-b3-traceid 1a6d063b2e392c71
act_server_1  | x-envoy-expected-rq-timeout-ms 15000
act_server_1  | user-agent grpc-c++/1.15.0 grpc-c/6.0.0 (linux; chttp2; glider)
act_server_1  | x-forwarded-proto http
act_server_1  | x-envoy-decorator-operation checkAvailability
act_server_1  | server-family ActService
act_server_1  | x-b3-spanid 25e2a308dbb1a295
```

##### 业务侧 Inbound
在envoy正常工作下，现在业务已经能接收到其转发的请求了。请求体中带了span-context，即构造一个SpanContext需要的信息在HTTP header中。如果是普通http则直接header中可取。我们业务是grpc请求，在信息在metadata里。
* 如何在收到rpc请求或者发起rpc调用时，进行合适的上报追踪信息？ 答案是 [grpc 拦截器(interceptor)](https://medium.com/@shijuvar/writing-grpc-interceptors-in-go-bf3e7671fe48)。
	* [client 拦截](https://grpc.io/grpc/python/grpc.html#client-side-interceptor)
	* [server拦截](https://grpc.io/grpc/python/grpc.html#service-side-interceptor)

自己实现并不复杂，网上也有人进行了[封装](https://github.com/rnburn/grpc-opentracing)，最后实际代码很简单：
```python
from grpc_opentracing import open_tracing_server_interceptor
from grpc_opentracing.grpcext import intercept_server

...
   # init a tracer
    config = Config(
        config={
            'sampler': {
                'type': 'const',
                'param': 1,
            },
            'local_agent': {
                'reporting_host': 'jaeger'
            },
            'logging': True,
            'tags': {
                'first_name': 'lin',
                'second_name': 'kevin'
            }
        }, service_name='act1', validate=True)

    tracer = config.initialize_tracer()
    tracer_interceptor = open_tracing_server_interceptor(tracer)
    server = grpc.server(futures.ThreadPoolExecutor( max_workers=10))
    
    # 设置tracing拦截器
    server = intercept_server(server, tracer_interceptor)

	# 现在接收的所有grpc请求都将会打一条operation=act1的日志。
```
* 注，这里本地不部署jaeger-agent,而直接将上报发给`reporting_host`（见上config配置）

除了找到合适的上报时机，还需要上报正确的内容，这就涉及到grpc本身协议机制（如metadata）和opentracing api的使用了。上面加上后，会发现tracing并没有沿着预想的一路下来，而是另建了一个新的Span，这是为什么呢？

回想一下整个过程:  grpc request->server interceptor->tracer？？
```python
class OpenTracingServerInterceptor(grpcext.UnaryServerInterceptor,
                                   grpcext.StreamServerInterceptor):
	...
    def _start_span(self, servicer_context, method):
        span_context = None
        error = None
        metadata = servicer_context.invocation_metadata() #这里带的headers信息
        try:
            if metadata:
                span_context = self._tracer.extract(
                    opentracing.Format.HTTP_HEADERS, dict(metadata)) #注意使用HTTP_HEADERS这个format去提取metadata
        except 
	        ...
        ...
        span = self._tracer.start_span(
            operation_name=method, child_of=span_context, tags=tags) #使用child_of创建子span
        ...
        return span
```
逻辑看起来清晰。而之所以未能继承自header的数据来追踪，问题显然是来自`span_context = self._tracer.extract(opentracing.Format.HTTP_HEADERS, dict(metadata))` 这段。深入extract()源码（jaeger的实现）后可见：
```python
  def extract(self, format, carrier):
        codec = self.codecs.get(format, None)
        if codec is None:
            raise UnsupportedFormatException(format)
        return codec.extract(carrier)
```
这里解码器codecs是可扩展的多种类型，而jaeger在`__init__`中是这样初始化：
```python
   self.codecs = {
            Format.TEXT_MAP: TextCodec(
                url_encoding=False,
                trace_id_header=trace_id_header,
                baggage_header_prefix=baggage_header_prefix,
                debug_id_header=debug_id_header,
            ),
            Format.HTTP_HEADERS: TextCodec(
                url_encoding=True,
                trace_id_header=trace_id_header,
                baggage_header_prefix=baggage_header_prefix,
                debug_id_header=debug_id_header,
            ),
            Format.BINARY: BinaryCodec(),
            ZipkinSpanFormat: ZipkinCodec(),
        }
        if extra_codecs:
            self.codecs.update(extra_codecs)
```
问题基本浮出水面，Format.HTTP_HEADERS被注册成TextCodec(...)的实例，这个并不能解B3-header系列的头部，所以codec.extract将返回None，于是没有和parent span串联。

最后利用extra_codecs->propagation->config['proopagation']='b3'来解决问题，即在初始化Tracer时，传入的config添加` 'propagation': 'b3'`即可。与此相关代码：
```python
  @property
    def propagation(self):
        propagation = self.config.get('propagation')
        if propagation == 'b3':
            # replace the codec with a B3 enabled instance
            return {Format.HTTP_HEADERS: B3Codec()}
        return {}
```

##### 业务侧 Outbound
业务侧的Outboud（向外部的其它请求）一般需要带上来源span，这样才能继续跟踪调用。有前面的基础后，在发起grpc请求前，创建一个client_interceptor，大概代码形如：
```python
from grpc_opentracing import open_tracing_client_interceptor
from grpc_opentracing.grpcext import intercept_channel

...
def AnGrpcCall(self, request, context):
    tracer_client_interceptor = open_tracing_client_interceptor(self_.tracer, active_span_source=ActiveSpanSource(self._tracer, context.invocation_metadata())
    channel = intercept_channel(channel, tracer_client_interceptor)
```
这里有个参数active_span_source，需要自行实现返回源Span。我们可以通过grpc调用时的context来产生一个。
```python
class ActiveSpanSource(six.with_metaclass(abc.ABCMeta)):
    """Provides a way to access an the active span."""
    def __init__(self, tracer, metadata):
        self._tracer = tracer
        self._metadata = metadata

    def get_active_span(self):
        """Identifies the active span.

    Returns:
      An object that implements the opentracing.Span interface.
    """
        span_ctx = None
        try:
            if self._metadata:
                span_ctx = self._tracer.extract(
                    opentracing.Format.HTTP_HEADERS, dict(self._metadata))
        except (opentracing.UnsupportedFormatException,
                opentracing.InvalidCarrierException,
                opentracing.SpanContextCorruptedException) as e:
            logging.exception(
                'tracer.extract() failed')
            return None
        return Span(span_ctx, self._tracer, 'Outbound')
```
至此，出去的所有调用都继承了来源的Span，整个链路即可串联。

##### 结果展示
![Alt text](/assets/201811/istio-grpc-tracing-show.png)


#### 附录：API纵览
* jaeger_client(以python模块举例）
	* Tracer模块
		* init(service_name, reporter, sampler, ...)
		* start_span(operation_name, child_of, references, tags, start_time)
		* inject(span_context, format, carrier)
		* extract(format, carrier)
		* close()
		* report_span(span)
		* random_id()
		* is_debug_allowed()
	* Config模块
		* init(config, metrics, service_name,...)
		* @service_name
		* @logging()
		* @trace_id_header()
		* @sampler
		* @tags
		* initialize_tracer(io_loop)
		* new_tracer()
		* create_tracer(reporter, sampler,...)
	* Span模块
		* init(context, tracer, operation_name, tags, start_time)
		* set_operation_name(name)
		* finish(finish_time)
		* set_tag(key, value) -- append
		* log_kv(key_values, timestamp)
		* set_baggage_item(key, value)
		* get_baggage_item(key)
		* is_sampled()
		* is_rpc()
		* is_rpc_client()
		* @trace_id
		* @span_id
		* @parent_id
		* @flags
	* SpanContext模块
		* init(trace_id, span_id, parent_id, flags, baggage, debug_id)
		* @baggage
		* with_baggage_item(key, value) -- return SpanContext
		* @has_trace
		* @debug_id
		* with_debug_id(debug_id) -- return SpanContext
	* Sampler模块
		* ConstSampler
		* ProbabilisticSampler
		* RateLimitingSampler
		* RemoteControlledSampler

### 参考
* [opentracing规格说明](https://opentracing.io/specification/)
* [使用Istio分布式跟踪应用程序](http://www.servicemesher.com/blog/distributed-tracing-istio-and-your-applications/)
* [openzipkin github](https://github.com/openzipkin/b3-propagation)
