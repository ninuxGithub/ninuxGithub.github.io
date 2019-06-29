---
title: java lambda 接口的使用
author: ninuxGithub
layout: post
date: 2019-6-29 09:38:15
description: "java lambda 接口的使用"
tag: java
---


## 目的
    有一种场景 比如我们需要根据特定的条件 从集合里面 获取符合特定条件的记录 过滤记录 ； lambda 为我们提供了更好的机制
    当然我们可以采用filter , 但是这个地方更多的是想要使用一些接口函数；
    
    Consumer  Function Predicate Supplier;
    
    名称	参数	返回值	实例
    Consumer	有	无	Iterable上的forEach方法
    Function	有	有	Optional的map方法
    Predicate	有	有(bool)	Optional的filter方法
    Supplier	无	有	懒加载、惰性求值、Stream的generate(静态)
    
```java
public class Demo {


    public static void main(String[] args) {
        new Demo().show(2, () -> {
            return "lambda interface call";
        });

        //Supplier get 来做生产
        int[] arr = {90, 2, 77, -1, 100};
        int max = getMax(() -> {
            int temp = 0;
            for (int e : arr) {
                if (e > temp) {
                    temp = e;
                }
            }
            return temp;
        });
        System.out.println("max element : " + max);


        //Consumer 消费
        consum("java", (name) -> {
            System.out.println(new StringBuilder(name).reverse());
        });


        //分别指向2个方法 name为原始的入参  java
        consums("java", (name) -> {
            System.out.println(name + "111111111111111");
        }, (name) -> {
            System.out.println(name + "2222222222222");
        });

        //Predict 来bool判断
        boolean checkString = checkString("java", (str) -> {
            return str.length() > 5;
        });
        System.out.println(checkString);




        //Function
        changeType("12",(s)->{
            return Integer.valueOf(s);
        });

        //第一次将字符串12 转为了12整数 ，然后将12整数和java doc进行拼接
        changeTypes("12",(s)->{
            return Integer.valueOf(s);
        },(s)->{
            //integer
            System.out.println(s.getClass().getName());
            return s + "java doc";
        });


    }


    public static void changeType(String s , Function<String,Integer> function){
        int in = function.apply(s);
        System.out.println(in);
    }
    public static void changeTypes(String s , Function<String,Integer> function, Function<Integer,String> function2){
        String in = function.andThen(function2).apply(s);
        System.out.println(in);
    }



    public static boolean checkString(String s, Predicate<String> predict) {
        return predict.test(s);
    }

    public static void consum(String name, Consumer<String> consumer) {
        consumer.accept(name);
    }

    public static void consums(String name, Consumer<String> consumer, Consumer<String> consumer2) {
        consumer.andThen(consumer2).accept(name);
    }


    public static int getMax(Supplier<Integer> supplier) {
        return supplier.get();
    }

    public void show(int level, MessageBuilder messageBuilder) {
        System.out.println(level + " -- " + messageBuilder.buildMessage());
    }

}
```   


    对条件的过滤   
    
```java
public class PredicateDemo {

    public static void main(String[] args) {

        //获取age>25的学生集合
        System.out.println(getTargetList(Student.list, (s) -> {
            return s.getAge() > 25;
        }));

        //获取age>25 或者  名字包含 java 的学生集合
        System.out.println(getTargetListOr(Student.list, (s) -> {
            return s.getAge() > 25;
        }, (s) -> {
            return s.getName().contains("java");
        }));


        //获取age>25 并且  名字包含 java 的学生集合


        //获取age>=25岁的学生
        System.out.println(getTargetListNagtive(Student.list, (s) -> {
            return s.getAge() < 25;
        }));

    }


    public static List<Student> getTargetList(List<Student> list, Predicate<Student> predicate) {
        List<Student> result = new ArrayList<>();
        for (Student s : list) {
            if (predicate.test(s)) {
                result.add(s);
            }
        }
        return result;
    }

    public static List<Student> getTargetListNagtive(List<Student> list, Predicate<Student> predicate) {
        List<Student> result = new ArrayList<>();
        for (Student s : list) {
            if (predicate.negate().test(s)) {
                result.add(s);
            }
        }
        return result;
    }

    public static List<Student> getTargetListOr(List<Student> list, Predicate<Student> predicate, Predicate<Student> predicate2) {
        List<Student> result = new ArrayList<>();
        for (Student s : list) {
            if (predicate.or(predicate2).test(s)) {
                result.add(s);
            }
        }
        return result;
    }

    public static List<Student> getTargetListAnd(List<Student> list, Predicate<Student> predicate, Predicate<Student> predicate2) {
        List<Student> result = new ArrayList<>();
        for (Student s : list) {
            if (predicate.and(predicate2).test(s)) {
                result.add(s);
            }
        }
        return result;
    }

}


public class Student {
    private int age;
    private String name;


    static List<Student> list = new ArrayList<Student>() {{
        add(new Student(18, "java"));
        add(new Student(23, "zs"));
        add(new Student(45, "ls"));
        add(new Student(22, "ww"));
        add(new Student(19, "zl"));
        add(new Student(32, "wm"));
        add(new Student(34, "jr"));
    }};

    public Student() {
    }

    public Student(int age, String name) {
        this.age = age;
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "Student{" +
                "age=" + age +
                ", name='" + name + '\'' +
                '}';
    }
}

``` 