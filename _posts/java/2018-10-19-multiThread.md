---
title: 任务切分采用多线程执行
author: ninuxGithub
layout: post
date: 2018-10-19 16:57:28 
description: "任务切分采用多线程执行"
tag: java
---
    
直接上代码:任务进行切分采用多线程执行
```java

//任务切分的数量
public static final int taskSize = 4;		
	
//线程池
public final static ExecutorService pool = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());
	

public void multiThread2SaveJson(String methodName,Object targetBean) {
        //初始化数据
		List<Map<String,Object>> secuMainList = initData();
		int len = secuMainList.size();
		int avg = len / taskSize;
		List<Future<Void>> futureList = new ArrayList<>();		
		List<Map<String,Object>> avgList = null;
		//任务切分
		if(len>=taskSize) {
			for (int k = 0; k < taskSize; k++) {
				avgList = (k != taskSize - 1) ? secuMainList.subList(k * avg, (k + 1) * avg): secuMainList.subList(k * avg, len);
				Future<Void> future = pool.submit(callbackFunc(avgList, methodName,targetBean));
				futureList.add(future);
			}
		}else {
			avgList = secuMainList;
			Future<Void> future = pool.submit(callbackFunc(avgList,methodName,targetBean));
			futureList.add(future);
		}
		
		//callable产生回调
		for (int k = 0; k < taskSize; k++) {
			try {
				futureList.get(k).get();
			} catch (InterruptedException | ExecutionException e) {
				e.printStackTrace();
			}
		}
		
		//关闭池
		pool.shutdown();
	}
	
	/**
	*这个例子是没有返回值的， 如果有返回值 需要加入对应的泛型 
    */
	private  Callable<Void> callbackFunc(List<Map<String,Object>> avgList,String targetMethod, Object thisObj) {
		Callable<Void> call = new Callable<Void>() {
			@Override
			public Void call() throws Exception {
				Class<?> clazz = thisObj.getClass();
				
				Method[] methods = clazz.getMethods();
				boolean methodDefined = false;
				for (int i = 0; i < methods.length; i++) {
					String methodName = methods[i].getName();
					if(methodName.equals(targetMethod)) {
						methodDefined = true; 
						break;
					}
				}				
				if(methodDefined) {
					Method declaredMethod = clazz.getDeclaredMethod(targetMethod, List.class);
					declaredMethod.invoke(thisObj, avgList);
					System.out.println(Thread.currentThread().getName() + "多线程正在调用"+targetMethod);
				}else {
					System.out.println(targetMethod +"方法在"+thisObj.getClass().getName()+"中没有定义这个方法");
				}				
				return null;
			}
		};
		return call;
	}
```
    
    
    