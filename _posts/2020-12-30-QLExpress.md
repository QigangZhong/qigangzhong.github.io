---
layout: post
title:  "QLExpress"
categories: tools
tags: tools
author: steve
---

* content
{:toc}










## getting started

[QLExpress](https://github.com/alibaba/QLExpress)

## example

### maven依赖

```xml
<dependencies>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>QLExpress</artifactId>
        <version>3.2.0</version>
    </dependency>

    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.8.2</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### hello world

```java
import com.ql.util.express.DefaultContext;
import com.ql.util.express.ExpressRunner;
import org.junit.Assert;
import org.junit.Test;

public class HelloWorldTest {

    @Test
    public void helloTest() throws Exception {
        ExpressRunner runner = new ExpressRunner();
        DefaultContext<String, Object> context = new DefaultContext<>();
        context.put("a", 1);
        context.put("b", 2);
        String express = "a+b";
        Object r = runner.execute(express, context, null, true, false);
        System.out.println(r);
        Assert.assertEquals(3, r);
    }

    /**
     * 循环操作符测试
     * @throws Exception if any
     */
    @Test
    public void operateLoopTest() throws Exception {
        final String express = "int n=10;" +
                "int sum=0;int i = 0;" +
                "for(i=0;i<n;i++){" +
                "sum=sum+i;" +
                "}" +
                "return sum;";
        ExpressRunner runner = new ExpressRunner();
        DefaultContext<String, Object> context = new DefaultContext<>();
        Object r = runner.execute(express, context, null, true, false);
        System.out.println(r);
        Assert.assertEquals(45, r);
    }

    /**
     * 三目运算符测试
     * 备注：测试不通过
     * @throws Exception if any
     */
    @Test
    public void logicalTernaryOperationsTest() throws Exception {
        final String express =
                "a=2; b=1; m = a>b?a:b; return m;";
        ExpressRunner runner = new ExpressRunner();
        DefaultContext<String, Object> context = new DefaultContext<>();
        Object r = runner.execute(express, context, null, true, false);
        System.out.println(r);
        Assert.assertEquals(2, r);
    }

    /**
     * 自定义函数测试
     * @throws Exception if any
     */
    @Test
    public void defineFunctionTest() throws Exception {
        final String express = "function add(int a,int b){\n" +
                "  return a+b;\n" +
                "};\n" +
                "\n" +
                "function sub(int a,int b){\n" +
                "  return a - b;\n" +
                "};\n" +
                "\n" +
                "a=10;\n" +
                "return add(a,4) + sub(a,9);";
        ExpressRunner runner = new ExpressRunner();
        DefaultContext<String, Object> context = new DefaultContext<>();
        Object r = runner.execute(express, context, null, true, false);
        System.out.println(r);
        Assert.assertEquals(15, r);
    }

    @Test
    public void replaceKeywordTest() throws Exception {
        ExpressRunner runner = new ExpressRunner();
        runner.addOperatorWithAlias("如果", "if", null);
        runner.addOperatorWithAlias("则", "then", null);
        runner.addOperatorWithAlias("否则", "else", null);
        DefaultContext<String, Object> context = new DefaultContext<>();
        final String express = "如果(1>2){ return 10;} 否则 {return 5;}";
        Object r = runner.execute(express, context, null, true, false);
        System.out.println(r);
        Assert.assertEquals(5, r);
    }
}
```

### 集合操作

```java
import com.ql.util.express.DefaultContext;
import com.ql.util.express.ExpressRunner;
import org.junit.Assert;
import org.junit.Test;

public class CollectionTest {
    /**
     * set 集合测试
     * 备注：集合的快捷写法
     * @throws Exception if any
     */
    @Test
    public void shorthandTest() throws Exception {
        ExpressRunner runner = new ExpressRunner(false,false);
        DefaultContext<String, Object> context = new DefaultContext<>();
        String express = "abc = NewMap(1:1,2:2); return abc.get(1) + abc.get(2);";
        Object r = runner.execute(express, context, null, false, false);
        Assert.assertEquals(3, r);
        express = "abc = NewList(1,2,3); return abc.get(1)+abc.get(2)";
        r = runner.execute(express, context, null, false, false);
        Assert.assertEquals(5, r);
        express = "abc = [1,2,3]; return abc[1]+abc[2];";
        r = runner.execute(express, context, null, false, false);
        Assert.assertEquals(5, r);
    }

    /**
     * 遍历测试
     *
     */
    @Test
    public void foreachTest() throws Exception {
        ExpressRunner runner = new ExpressRunner(false,false);
        DefaultContext<String, Object> context = new DefaultContext<>();
        String express =
                "  Map map = new HashMap();\n" +
                        "  map.put(\"a\", \"a_value\");\n" +
                        "  map.put(\"b\", \"b_value\");\n" +
                        "  keySet = map.keySet();\n" +
                        "  objArr = keySet.toArray();\n" +
                        "  for (i=0;i<objArr.length;i++) {\n" +
                        "  key = objArr[i];\n" +
                        "   System.out.println(map.get(key));\n" +
                        "  }";
        Object r = runner.execute(express, context, null, false, false);
        System.out.println(r);
    }
}
```

### macro宏（表达式别名）

```java
import com.ql.util.express.ExpressRunner;
import com.ql.util.express.IExpressContext;
import org.junit.Assert;
import org.junit.Test;

public class MacroTest {
    @Test
    public void macroTest() throws Exception {
        ExpressRunner runner = new ExpressRunner();
        runner.addMacro("计算平均成绩", "(语文+数学+英语)/3.0");
        runner.addMacro("是否优秀", "计算平均成绩>90");
        IExpressContext<String, Object> context = new DefaultContext<>();
        context.put("语文", 88);
        context.put("数学", 99);
        context.put("英语", 95);
        Boolean result = (Boolean) runner.execute("是否优秀", context, null, false, false);
        System.out.println(result);
        Assert.assertTrue(result);
    }
}
```

```java
//调用java 对象测试
import com.qigang.rule.model.User;
import com.ql.util.express.DefaultContext;
import com.ql.util.express.ExpressRunner;
import com.ql.util.express.IExpressContext;
import org.junit.Assert;
import org.junit.Test;

/**
 * <p> 对象测试 </p>
 *
 * 备注：例子来自官方例子
 */
public class ObjectTest {

    @Test
    public void test1() throws Exception {
        String exp = "import com.qigang.rule.model.User;" +
                "User cust = new User(1);" +
                "cust.setName(\"小强\");" +
                "return cust.getName();";
        ExpressRunner runner = new ExpressRunner();
        String r = (String) runner.execute(exp, null, null, false, false);
        System.out.println(r);
        Assert.assertEquals("操作符执行错误", "小强", r);
    }

    @Test
    public void test2() throws Exception {
        String exp = "cust.setName(\"小强\");" +
                "return cust.getName();";
        IExpressContext<String, Object> expressContext = new DefaultContext<>();
        expressContext.put("cust", new User(1));
        ExpressRunner runner = new ExpressRunner();
        String r = (String) runner.execute(exp, expressContext, null, false, false);
        Assert.assertEquals("操作符执行错误", "小强", r);
    }

    @Test
    public void test3() throws Exception {
        String exp = "首字母大写(\"abcd\")";
        ExpressRunner runner = new ExpressRunner();
        runner.addFunctionOfClassMethod("首字母大写", User.class.getName(), "firstToUpper", new String[]{"String"}, null);
        String r = (String) runner.execute(exp, null, null, false, false);
        System.out.println(r);
        Assert.assertEquals("操作符执行错误", "Abcd", r);
    }

    /**
     * 使用别名
     *
     * @throws Exception if any
     */
    @Test
    public void testAlias() throws Exception {
        String exp = "cust.setName(\"小强\");" +
                "定义别名 custName cust.name;" +
                "return custName;";
        IExpressContext<String, Object> expressContext = new DefaultContext<>();
        expressContext.put("cust", new User(1));
        ExpressRunner runner = new ExpressRunner();
        //
        runner.addOperatorWithAlias("定义别名", "alias", null);
        //执行表达式，并将结果赋给r
        String r = (String) runner.execute(exp, expressContext, null, false, false);
        System.out.println(r);
        Assert.assertEquals("操作符执行错误", "小强", r);
    }

    /**
     * 使用宏
     *
     * @throws Exception if any
     */
    @Test
    public void testMacro() throws Exception {
        String exp = "cust.setName(\"小强\");" +
                "定义宏 custName {cust.name};" +
                "return custName;";
        IExpressContext<String, Object> expressContext = new DefaultContext<>();
        expressContext.put("cust", new User(1));
        ExpressRunner runner = new ExpressRunner();
        runner.addOperatorWithAlias("定义宏", "macro", null);
        String r = (String) runner.execute(exp, expressContext, null, false, false);
        System.out.println(r);
        Assert.assertEquals("操作符执行错误", "小强", r);
    }

}
```

### 使用java对象

```java
import com.qigang.rule.model.User;
import com.ql.util.express.DefaultContext;
import com.ql.util.express.ExpressRunner;
import com.ql.util.express.IExpressContext;
import org.junit.Assert;
import org.junit.Test;

/**
 * <p> 对象测试 </p>
 *
 * 备注：例子来自官方例子
 */
public class ObjectTest {
    @Test
    public void test1() throws Exception {
        String exp = "import com.qigang.rule.model.User;" +
                "User cust = new User(1);" +
                "cust.setName(\"小强\");" +
                "return cust.getName();";
        ExpressRunner runner = new ExpressRunner();
        String r = (String) runner.execute(exp, null, null, false, false);
        System.out.println(r);
        Assert.assertEquals("操作符执行错误", "小强", r);
    }

    @Test
    public void test2() throws Exception {
        String exp = "cust.setName(\"小强\");" +
                "return cust.getName();";
        IExpressContext<String, Object> expressContext = new DefaultContext<>();
        expressContext.put("cust", new User(1));
        ExpressRunner runner = new ExpressRunner();
        String r = (String) runner.execute(exp, expressContext, null, false, false);
        Assert.assertEquals("操作符执行错误", "小强", r);
    }

    @Test
    public void test3() throws Exception {
        String exp = "首字母大写(\"abcd\")";
        ExpressRunner runner = new ExpressRunner();
        runner.addFunctionOfClassMethod("首字母大写", User.class.getName(), "firstToUpper", new String[]{"String"}, null);
        String r = (String) runner.execute(exp, null, null, false, false);
        System.out.println(r);
        Assert.assertEquals("操作符执行错误", "Abcd", r);
    }

    /**
     * 使用别名
     *
     * @throws Exception if any
     */
    @Test
    public void testAlias() throws Exception {
        String exp = "cust.setName(\"小强\");" +
                "定义别名 custName cust.name;" +
                "return custName;";
        IExpressContext<String, Object> expressContext = new DefaultContext<>();
        expressContext.put("cust", new User(1));
        ExpressRunner runner = new ExpressRunner();
        //
        runner.addOperatorWithAlias("定义别名", "alias", null);
        //执行表达式，并将结果赋给r
        String r = (String) runner.execute(exp, expressContext, null, false, false);
        System.out.println(r);
        Assert.assertEquals("操作符执行错误", "小强", r);
    }

    /**
     * 使用宏
     *
     * @throws Exception if any
     */
    @Test
    public void testMacro() throws Exception {
        String exp = "cust.setName(\"小强\");" +
                "定义宏 custName {cust.name};" +
                "return custName;";
        IExpressContext<String, Object> expressContext = new DefaultContext<>();
        expressContext.put("cust", new User(1));
        ExpressRunner runner = new ExpressRunner();
        runner.addOperatorWithAlias("定义宏", "macro", null);
        String r = (String) runner.execute(exp, expressContext, null, false, false);
        System.out.println(r);
        Assert.assertEquals("操作符执行错误", "小强", r);
    }
}

public class User {

    /**
     * 标识
     */
    private long id;

    /**
     * 名称
     */
    private String name;

    /**
     * 年龄
     */
    private int age;

    public User(long id){
        this.id = id;
    }

    public long getId() {
        return id;
    }
    public void setId(long id) {
        this.id = id;
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public int getAge() {
        return age;
    }
    public void setAge(int age) {
        this.age = age;
    }

    /**
     * 首字母大写
     * @param value 字符串
     * @return 转换后的信息
     */
    public static String firstToUpper(String value){
        if(StringUtils.isBlank(value))
            return "";
        value = StringUtils.trim(value);
        String f = StringUtils.substring(value,0,1);
        String s = "";
        if(value.length() > 1){
            s = StringUtils.substring(value,1);
        }
        return f.toUpperCase() + s;
    }
}
```

### 自定义操作符

```java
import com.qigang.rule.operator.JoinOperator;
import com.ql.util.express.DefaultContext;
import com.ql.util.express.ExpressRunner;
import org.junit.Assert;
import org.junit.Test;
import java.util.Arrays;

public class OperatorTest {
    @Test
    public void addOperatorTest() throws Exception {
        ExpressRunner runner = new ExpressRunner();
        DefaultContext<String, Object> context = new DefaultContext<>();
        runner.addOperator("join", new JoinOperator());
        Object r = runner.execute("1 join 2 join 3", context, null, false, false);
        System.out.println(r);
        Assert.assertEquals(Arrays.asList(1,2,3), r);
    }
    @Test
    public void replaceOperatorTest() throws Exception {
        ExpressRunner runner = new ExpressRunner();
        DefaultContext<String, Object> context = new DefaultContext<>();
        runner.replaceOperator("+", new JoinOperator());
        Object r = runner.execute("1 + 2 + 3", context, null, false, false);
        System.out.println(r);
        Assert.assertEquals(Arrays.asList(1,2,3), r);
    }
    @Test
    public void addFunctionTest() throws Exception {
        ExpressRunner runner = new ExpressRunner();
        DefaultContext<String, Object> context = new DefaultContext<>();
        runner.addFunction("join",new JoinOperator());
        Object r = runner.execute("join(1, 2, 3)", context, null, false, false);
        System.out.println(r);
        Assert.assertEquals(Arrays.asList(1,2,3), r);
    }
}

import com.ql.util.express.Operator;
public class JoinOperator extends Operator {
    private static final long serialVersionUID = 5653601029469696306L;
    @Override
    public Object executeInner(Object[] objects) {
        java.util.List result = new java.util.ArrayList();

        for (Object object : objects) {
            if (object instanceof java.util.List) {
                result.addAll(((java.util.List) object));
            } else {
                result.add(object);
            }
        }
        return result;
    }
}
```

### 小数精度设置

```java
import com.ql.util.express.DefaultContext;
import com.ql.util.express.ExpressRunner;
import com.ql.util.express.IExpressContext;
import org.junit.Test;

public class PreciseTest {
    /**
     * 测试精度设置
     * @throws Exception if any
     */
    @Test
    public void testPercise() throws Exception {
        ExpressRunner runner = new ExpressRunner(true, false);
        IExpressContext<String,Object> expressContext = new DefaultContext<>();
        String expression ="12.3/3";
        Object result = runner.execute(expression, expressContext, null, false, false);
        System.out.println(result);
    }
}
```

### 短路测试

```java
import com.ql.util.express.DefaultContext;
import com.ql.util.express.ExpressRunner;
import com.ql.util.express.IExpressContext;
import org.junit.Test;
import java.util.ArrayList;
import java.util.List;

/**
 * <p> 短路运算符测试 </p>
 */
public class ShortcutTest {

    private ExpressRunner runner = new ExpressRunner();

    private void initial() throws Exception{
        runner.addOperatorWithAlias("小于","<","$1 小于 $2 不满足期望");
        runner.addOperatorWithAlias("大于",">","$1 大于 $2 不满足期望");
    }

    private boolean calculateLogicTest(String expression, IExpressContext<String, Object> expressContext, List<String> errorInfo) throws Exception {
        return (Boolean)runner.execute(expression, expressContext, errorInfo, true, false);
    }

    /**
     * 测试非短路逻辑,并且输出出错信息
     * @throws Exception if any
     */
    @Test
    public void testShortCircuit() throws Exception {
        runner.setShortCircuit(false);
        IExpressContext<String,Object> expressContext = new DefaultContext<>();
        expressContext.put("违规天数", 100);
        expressContext.put("虚假交易扣分", 11);
        expressContext.put("VIP", false);
        List<String> errorInfo = new ArrayList<>();
        initial();
        String expression ="2 小于 1 and (违规天数 小于 90 or 虚假交易扣分 小于 12)";
        boolean result = calculateLogicTest(expression, expressContext, errorInfo);
        System.out.println(result);
        showErrorInfo(result, errorInfo);
    }

    /**
     * 测试非短路逻辑,并且输出出错信息
     * @throws Exception if any
     */
    @Test
    public void testNoShortCircuit() throws Exception {
        runner.setShortCircuit(false);
        IExpressContext<String,Object> expressContext = new DefaultContext<>();
        expressContext.put("违规天数", 100);
        expressContext.put("虚假交易扣分", 11);
        expressContext.put("VIP", false);
        List<String> errorInfo = new ArrayList<>();
        initial();
        String expression ="2 小于 1 and (违规天数 小于 90 or 虚假交易扣分 小于 12)";
        boolean result = calculateLogicTest(expression, expressContext, errorInfo);
        showErrorInfo(result, errorInfo);
    }

    /**
     * 展现错误信息
     * @param result 结果
     * @param errorList 错误列表
     */
    private void showErrorInfo(boolean result, List<String> errorList) {
        if(result){
            System.out.println("result is success!");
        }else{
            System.out.println("result is fail!");
            for(String error : errorList){
                System.out.println(error);
            }
        }
    }
}
```

### 绑定java类中的方法

```java
import com.qigang.rule.bind.BindObjectMethod;
import com.ql.util.express.DefaultContext;
import com.ql.util.express.ExpressRunner;
import org.junit.Test;

public class BindMethodTest {
    @Test
    public void bindObjectMethodTest() throws Exception {
        ExpressRunner runner = new ExpressRunner();
        DefaultContext<String, Object> context = new DefaultContext<>();

        runner.addFunctionOfClassMethod("取绝对值", Math.class.getName(), "abs",
                new String[] { "double" }, null);
        runner.addFunctionOfClassMethod("转换为大写", BindObjectMethod.class.getName(),
                "upper", new String[] { "String" }, null);
        runner.addFunctionOfServiceMethod("打印", System.out, "println",new String[] { "String" }, null);
        runner.addFunctionOfServiceMethod("contains", new BindObjectMethod(), "anyContains",
                new Class[] { String.class, String.class }, null);
        String exp = "取绝对值(-100);转换为大写(\"hello world\");打印(\"你好吗？\");contains(\"helloworld\",\"aeiou\")";
        Object r = runner.execute(exp, context, null, false, false);
        System.out.println(r);
    }
}

public class BindObjectMethod {
    /**
     * 大写
     * @param abc 字符串
     * @return 转换后
     */
    public static String upper(String abc) {
        return abc.toUpperCase();
    }

    /**
     * 任何包含
     * @param str 字符串
     * @param searchStr 查询字符串
     * @return 是否包含
     */
    public boolean anyContains(String str, String searchStr) {

        char[] s = str.toCharArray();
        for (char c : s) {
            if (searchStr.contains(c + "")) {
                return true;
            }
        }
        return false;
    }
}
```



## 与其他规则引擎的比较

* [Drools](https://github.com/kiegroup/drools)
* [easy-rules](https://github.com/j-easy/easy-rules)
* [urule](https://github.com/youseries/urule)

## 参考

[QLExpress官方示例](https://github.com/alibaba/QLExpress/tree/master/src/test/java/com/ql/util/express/example)

[规则引擎：大厂营销系统资格设计全解](https://mp.weixin.qq.com/s/TiSEYDdODiuVyBOWpOMzRA)
