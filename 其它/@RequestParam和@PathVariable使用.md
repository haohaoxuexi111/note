# @RequestParam和@PathVariable使用

### 一.@RequestParam

@RequestParam用于(普通url)获取参数，可获取?username="sss"这种？后面的参数值

#### 方式一:

```java
//在url中输入:localhost:8080/**/?userName=zhangsan
//请求中包含username参数（如/requestparam1?userName=zhang），则自动传入。
public String queryUserName(@RequestParam String userName)
```

除了简单地定义{username}变量，还可以定义正则表达式进行更精确地控制，定义语法是{变量名: 正则表达式}。[a-zA-Z0-9_]+是一个正则表达式，表示只能包含小写字母，大写字母，数字，下划线。如此设置URL变量规则后，不合法的URL则不会被处理，直接由SpringMVC框架返回404NotFound。

```java
@RequestMapping(value = "/user/{username: [a-zA-Z0-9]+}/blog/{blogId}")
```

#### 方式二:

处理参数名称不一致的情况，如果前台传入lid到后台，可给该参数设置别名为id==(也可将String类型换成int，会自动转换)==，用@RequestParam注解从请求参数中映射到控制器中的参数时，==控制器的参数一定要用对象类型或简单类型的包装类==。例如@RequestParam(value="lid") Integer id)不能写成@RequestParam(value="lid") int id)，==不能用简单int类型去接收请求中的整数==。因为，若请求中的对象为空，则int类型的参数不能接收空对象，int类型的参数必须要有一个默认值的。 
若想用简单类型去接收请求中的值，需要赋值一个默认值，写成如下的形式：@RequestParam(value = "lid", required = false, defaultValue = "0") int id)

```java
@RequestMapping("/")
public String Demo1(@RequestParam(name="lid") String id){
    System.out.println("----"+id);
    return id;
}
控制台输出
----10
```

#### 特殊一:

但是在传递参数的时候如果是url?userName=zhangsan&userName=wangwu时，即两个同名参数，前台传递了两个一样的参数，可用如下方式:

```java
public String requestparam8(@RequestParam(value="userName") String []  userNames)
//或者是
public String requestparam8(@RequestParam(value="list") List<String> list)  
```

#### 特殊二: 

前端传参的URL： url = “${ctx}/main/mm/am/edit?Id=${Id}&name=${name}” 后端使用集合来接受参数，灵活性较好

```java
@RequestMapping("/edit")
public String edit(Model model, @RequestParam Map<String, Object> paramMap ) {
        long id = Long.parseLong(paramMap.get("id").toString());
        String name = paramMap.get("name").toString;
        return page("edit");
}
```

基本类型参数绑定容易出现的问题

@RequestParam绑定基本数据类型，若required属性为false（默认为true），且设置了defaultValue属性，没有问题；

@RequestParam绑定基本数据类型，若required属性为false（默认为true），且没有设置defaultValue属性，则当没有传该参数时，会报500（因为无法将null赋值给基本数据类型）

#### 二.@PathVariable

使用@PathVariable接收(Restful风格url)参数，参数值需要在url进行占位， 前端传参的URL：url = “${ctx}/main/mm/am/edit/${Id}/${name}”

```java
@RequestMapping("/edit/{id}/{name}")
public String edit(Model model, @PathVariable long id,@PathVariable String name) {
      return page("edit");
}
```

PathVariable 汉语意思是：路径变量，顾名思义，就是要获取一个url 地址中的一部分值，哪一部分呢？ RequestMapping 上说明了@RequestMapping(value="/edit/{id}/{name}"），我就是想获取你URL地址 /edit/ 的后面的那个 {id}以及{name}。

例如下列地址:http://localhost:8989/SSSP/emp/7

如果我需要获取emp后面的7，如下:

```java
@RequestMapping(value="/emp/{id}",method=RequestMethod.GET)  
public String edit(@PathVariable("id")Integer id,Map<String , Object>map){  
    Employee employee = employeeService.getEmployee(id);  
    List<Department> departments = departmentService.getAll();  
    map.put("employee", employee);  
    map.put("departments", departments);  
    return "emp/input";  
}
```

