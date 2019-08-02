---
title: SpringCloud Ribbon 原理分析
author: ninuxGithub
layout: post
date: 2019-8-1 14:59:55
description: "Ribbon 启动分析"
tag: spring-cloud
---

## 目的
    分析ribbon启动的过程是如何的；
    
    
    
## 源代码分析
    从服务的控制器调用service层里面的服务的时候， 我们会通过restTemplate来调用注册到eureka中心的服务，试着想想我们为什么在
    启用了ribbon之后就可以自动的做负载均衡了，我们什么额外的动作都没有去做； 我们只是引入了ribbon的依赖启动了ribbon的注解
    那么下面从现象到本质的分析；
    
    我们的项目通过配置负载均衡的规则和HystrixCommandAspect来进行了ribbon的自定义的一些配置
    并且在main方法入口加入了@RibbonClient @EnableHystrix 启用了ribbon,hystrix
    
    
    hystirx原理
        通过HystrixCommandAspect切面拦截所有加入了@HystrixCommand注解的方法， 提取注解中的方法， 在我们的目标方法执行的时候会提取到
        我们目标方法上面的注解的参数， 从而构成对象HystrixInvokable对象， 抽象的子类有HystrixCommand 内部有run , execute等方法
        也就是说我们的HystrixInvokable是一个可以执行的对象返回结果；如果出现了服务的超时会调用fallback;
        在hystrix进行debug的时候发现， 如果中间到某些代码有debug端点就会导致超时从而执行了fallback; 
        原因是代码中有关于线程中断的处理，如果有中断那么就认为执行hystrix失败了， 所以打断点hystrix执行的时候会失败；
        
    
```java
@Configuration
public class RibbonRuleConfig {
    
    @Bean
    public IRule ribbonRule(){
        return new WeightedResponseTimeRule();
    }

    @Bean
    public HystrixCommandAspect hystrixCommandAspect() {
        return new HystrixCommandAspect();
    }
}

class HystrixCommandAspect{
    
    @Around("hystrixCommandAnnotationPointcut() || hystrixCollapserAnnotationPointcut()")
    public Object methodsAnnotatedWithHystrixCommand(final ProceedingJoinPoint joinPoint) throws Throwable {
        Method method = getMethodFromTarget(joinPoint);
        Validate.notNull(method, "failed to get method from joinPoint: %s", joinPoint);
        if (method.isAnnotationPresent(HystrixCommand.class) && method.isAnnotationPresent(HystrixCollapser.class)) {
            throw new IllegalStateException("method cannot be annotated with HystrixCommand and HystrixCollapser " +
                    "annotations at the same time");
        }
        MetaHolderFactory metaHolderFactory = META_HOLDER_FACTORY_MAP.get(HystrixPointcutType.of(method));
        MetaHolder metaHolder = metaHolderFactory.create(joinPoint);
        //提取注解的fallback 构建对象
        HystrixInvokable invokable = HystrixCommandFactory.getInstance().create(metaHolder);
        ExecutionType executionType = metaHolder.isCollapserAnnotationPresent() ?
                metaHolder.getCollapserExecutionType() : metaHolder.getExecutionType();

        Object result;
        try {
            if (!metaHolder.isObservable()) {
                //通过commandExecutor进行执行之后返回结果，其实就是专为HystrixExecutable对象在调用execute()方法
                //execute 如何执行的呢？
                //debug 发现会进入到HystirxCommand queue()方法里面，大概的思想是任务会进入到队列里面返回一个future
                //toObservable()方法会添加一些类的监听到当前的任务上面
                //然后会为这个future对象添加一些监听 ，返回future
                //如果发现有出现了超时的异常的时候会执行fallback
                result = CommandExecutor.execute(invokable, executionType, metaHolder);
            } else {
                 //metaHolder是Observable类型的 是一个基于Rxjava 反应式编程的构想实现的事件驱动的方法；
                 //大概就是有个执行的任务，然后添加一个监视器，当这个任务执行的结果会形成个状态加入到call 然后执行对应的业务；
                result = executeObservable(invokable, executionType, metaHolder);
            }
        } catch (HystrixBadRequestException e) {
            throw e.getCause() != null ? e.getCause() : e;
        } catch (HystrixRuntimeException e) {
            throw hystrixRuntimeExceptionToThrowable(metaHolder, e);
        }
        return result;
    }
    
    //调用链如下
    
    //-->toObservable()
    
    //-->applyHystrixSemantics
    
    //-->executeCommandAndObserve
    
    
    //添加处理fallback监听的业务
    final Func1<Throwable, Observable<R>> handleFallback = new Func1<Throwable, Observable<R>>() {
        @Override
        public Observable<R> call(Throwable t) {
            circuitBreaker.markNonSuccess();
            Exception e = getExceptionFromThrowable(t);
            executionResult = executionResult.setExecutionException(e);
            if (e instanceof RejectedExecutionException) {
                return handleThreadPoolRejectionViaFallback(e);
            } else if (t instanceof HystrixTimeoutException) {
                return handleTimeoutViaFallback();
            } else if (t instanceof HystrixBadRequestException) {
                return handleBadRequestByEmittingError(e);
            } else {
                /*
                 * Treat HystrixBadRequestException from ExecutionHook like a plain HystrixBadRequestException.
                 */
                if (e instanceof HystrixBadRequestException) {
                    eventNotifier.markEvent(HystrixEventType.BAD_REQUEST, commandKey);
                    return Observable.error(e);
                }

                return handleFailureViaFallback(e);
            }
        }
    };
    
    //如果没有意外那么会进入的我们的目标方法， 进行方法的invoke
    /**
    * 在通过目标方法内部通过restTemplate发起请求的时候会被LoadBalancerInterceptor拦截
           通过loadbalancer.execute来执行我们的请求，实现客户端的负载均衡
           RibbonLoadBalancerClient进行请求的执行， 获取负载均衡器， 获取服务器的列表， 创建一个RibbonServer对象
           执行execute(serviceId, ribbonServer, request)
           T returnVal = request.apply(serviceInstance); -->LoadBalancerRequestFactory.createRequest 
           -->通过InteceptingClientHttpRequest.execute来发起请求,成功返回结果
    */
}


public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {

	@Override
	public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) throws IOException {
		final URI originalUri = request.getURI();
		String serviceName = originalUri.getHost();
		Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
		//拦截restTemplate的请求， 让我们的负载均衡器来处理执行请求
		return this.loadBalancer.execute(serviceName, requestFactory.createRequest(request, body, execution));
	}
}


/**
 * @author Spencer Gibb
 * @author Dave Syer
 * @author Ryan Baxter
 * @author Tim Ysewyn
 */
public class RibbonLoadBalancerClient implements LoadBalancerClient {


	@Override
	public ServiceInstance choose(String serviceId) {
		Server server = getServer(serviceId);
		if (server == null) {
			return null;
		}
		return new RibbonServer(serviceId, server, isSecure(server, serviceId),
				serverIntrospector(serviceId).getMetadata(server));
	}

	@Override
	public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
	    //根据服务的id获取负载均衡器
		ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
		//根据负载均衡策略 获取我们的服务器列表中的一个节点
		//loadBalancer.chooseServer("default");
		//不同的负载均衡器的choose方法不一样从而达到不同的效果的负载均衡策略
		//因为我们配置的是WeightedResponseTimeRule权重的负载均衡，最终还是会走这个负载均衡的策略
		Server server = getServer(loadBalancer);
		if (server == null) {
			throw new IllegalStateException("No instances available for " + serviceId);
		}
		
		//创建ribbonServer对象
		RibbonServer ribbonServer = new RibbonServer(serviceId, server, isSecure(server,
				serviceId), serverIntrospector(serviceId).getMetadata(server));

        //开始执行请求
		return execute(serviceId, ribbonServer, request);
	}

	@Override
	public <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException {
		Server server = null;
		if(serviceInstance instanceof RibbonServer) {
			server = ((RibbonServer)serviceInstance).getServer();
		}
		if (server == null) {
			throw new IllegalStateException("No instances available for " + serviceId);
		}

		RibbonLoadBalancerContext context = this.clientFactory
				.getLoadBalancerContext(serviceId);
		RibbonStatsRecorder statsRecorder = new RibbonStatsRecorder(context, server);

		try {
		    //apply会调用那个什么呢？  继续往下debug
			T returnVal = request.apply(serviceInstance);
			statsRecorder.recordStats(returnVal);
			return returnVal;
		}
		// catch IOException and rethrow so RestTemplate behaves correctly
		catch (IOException ex) {
			statsRecorder.recordStats(ex);
			throw ex;
		}
		catch (Exception ex) {
			statsRecorder.recordStats(ex);
			ReflectionUtils.rethrowRuntimeException(ex);
		}
		return null;
	}

}


public class AsyncLoadBalancerInterceptor implements AsyncClientHttpRequestInterceptor {

	private LoadBalancerClient loadBalancer;

	public AsyncLoadBalancerInterceptor(LoadBalancerClient loadBalancer) {
		this.loadBalancer = loadBalancer;
	}

	@Override
	public ListenableFuture<ClientHttpResponse> intercept(final HttpRequest request, final byte[] body,
			final AsyncClientHttpRequestExecution execution) throws IOException {
		final URI originalUri = request.getURI();
		String serviceName = originalUri.getHost();
		return this.loadBalancer.execute(serviceName,
            new LoadBalancerRequest<ListenableFuture<ClientHttpResponse>>() {
                @Override
                public ListenableFuture<ClientHttpResponse> apply(final ServiceInstance instance)throws Exception {
                    HttpRequest serviceRequest = new ServiceRequestWrapper(request,instance, loadBalancer);
                    //继续通过以下代理来实现我们的请求
                    return execution.executeAsync(serviceRequest, body);
                }

            });
	}
}


```    

    
    
    






    
    
    