---
title: androdid-序列化对单例模式的破坏
date: 2016-08-17 09:54:02
categories: android
tags: ["Android", "序列化与反序列化", "单例模式"]
---
# 一、 反序列化对Java单例的影响

##### 解决方案：让Singleton继承Serializable, 并实现  private Object readResolve();
    public class Singleton implement Serializable {
	 
	 	public static class Builder {
	 		public static Singleton instance = new Singleton();
	 	}
	 	
		public static Singleton getInstance(){
			return Builder.instance;
		}
		
		private Object readResolve(){
			retutn Builder.instance;
		}
	}

##### 原理解析
在反序列化时，ObjectInputStream 因为利用反射机制调用了 readObject --> readObject0 --> readOrdinary --> CheckResolve。在readOrdinady中调用了invokeReadResolve()，该方法使用反射机制创建新的对象，从而破坏了单例唯一性。
###[尊重版权](http://www.hollischuang.com/archives/1144)




####  二、 java单例模式的实现方式

#####1. 饿汉式
	public class Singleton {

    	private static Singleton = new Singleton();

    	private Singleton() {}

    	public static getSignleton(){
        	return singleton;
    	}
	}
#####2. 懒汉式
	public class Singleton {
		//注意volatiile的用法: 1. 保证在当前线程修改临界值之后在随后线程中起作用（可见性），及线程透明
		volatile的第二层语义是禁止指令重排序优化。
   		private static volatile Singleton singleton = null;
 
    	private Singleton(){}
 
    	public static Singleton getSingleton(){
    		if(singleton == null){}
        		synchronized (Singleton.class){
            		if(singleton == null){
                		singleton = new Singleton();
            		}
       			}
       		}

        	return singleton;
   		 }    
 
	}

#####3. 静态内部类
	public class Singleton {
		//因为静态内部类只会加载一次，所以保证了线程安全
	    private static class Holder {
	        private static Singleton singleton = new Singleton();
	    }
	 
	    private Singleton(){}
	 
	    public static Singleton getSingleton(){
	        return Holder.singleton;
	    }
	}

#####4. 枚举方式

	public enum Singleton {
	    INSTANCE;
	    private String name;
	    public String getName(){
	        return name;
	    }
	    public void setName(String name){
	        this.name = name;
	    }
	}
	
	使用枚举除了线程安全和防止反射强行调用构造器之外，还提供了自动序列化机制，防止反序列化的时候创建新的对象。因此，Effective Java推荐尽可能地使用枚举来实现单例。