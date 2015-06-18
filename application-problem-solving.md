# 教计算机程序解数学题

周末，看关于专家系统方面的书，其中有关于规则方面的内容，忽然就想，能不能模仿人的学习方式来提升计算机程序的计算能力呢？ 

 试想，一个小孩子，他一开始什么也不会，首先，你要告诉他什么是数字，然后告诉他什么是加、减；然后告诉他什么是乘、除，还要告诉他有乘、除要先计算乘除，然后又引入了括号说，有括号永远要先计算括号。如此，随着告诉他的技能越多，他的解题能力也就越强。  
于是就想着试验一下。  

**第一步，教计算机学习什么是数字。** 

下面的正则表达式，就是告诉“孩子”，数字就是前面可能有“-”号，当然也可能没有，接下来连续的数字0-9，组成的数字，后面可能还会有小数点开始加一堆0-9的数字，当然没有也没有关系。如此，它就算懂得认数字了。 

 
```
public final class MathNumber {  
    private MathNumber() {  
    }  
   
    public static String numberPattern = "[-]?[0-9]+([.][0-9]*)?";  
    public static Pattern pattern = Pattern.compile(numberPattern);  
   
    public static Matcher match(String string) {  
        Matcher match = pattern.matcher(string);  
        if (match.find()) {  
            return match;  
        }  
        throw new RuntimeException(string + " is not a number.");  
    }  
}  
```

**第二步就是告诉“孩子”，计算数学题的过程。**

如果两边有空格就忽略它，然后呢，看看是不是已经是一个数字了，如果已经是一个数字，那说明就算出结果了。如果不是，就从最高优先级找起，如果找就就计算。如果找不到，说明这个式子有问题，不是一个合法的数学式子。

```
public static String eval(String string) {  
        string = string.trim();  
        while (!isMathNumber(string)) {// 同一优先级的哪个先找到算哪个  
            System.out.println("求解算式：" + string);  
            boolean found = false;  
            for (MathInterface math : mathList) {  
                Matcher matcher = math.match(string);  
                if (matcher.find()) {  
   
                    String exp = matcher.group();  
                    String sig = "";  
                    if (exp.charAt(0) == '-' && matcher.start() != 0) {// 如果不是第一个数字，-号只能当运算符  
                        sig = "+";  
                    }  
                    System.out.println("发现算式：" + exp);  
                    String evalResult = math.eval(exp);  
                    string = string.substring(0, matcher.start()) + sig  
                            + evalResult + string.substring(matcher.end());  
                    System.out.println(exp + "计算结果为：" + evalResult + ",代回原式");  
                    found = true;  
                    break;  
                }  
            }  
            if (!found) {  
                throw new RuntimeException(string + " 不是合法的数学表达式");  
            }  
        }  
        return string;  
    }  
```

从现在开始，这孩子已经会解题思路了，不过他还是啥也不懂，他还不知道啥是加，减、乘、除啥的，没有办法，孩子笨，只要多教他了。  

下面就教他如何计算，加、减、乘、除、余、括号、指数。

```
addMathExpression(new Add());  
 addMathExpression(new Subtract());  
 addMathExpression(new Multiply());  
 addMathExpression(new Devide());  
 addMathExpression(new Minus());  
 addMathExpression(new Factorial());  
 addMathExpression(new Remainder());  
 addMathExpression(new Bracket());  
 addMathExpression(new Power());  
 Collections.sort(mathList, new MathComparator());  
```

由于大同小异，就里就只贴出来加法和括号的实现方式。  

加法实现，它的优先级是1，它是由两个数字中间加一个“+”号构成，数字和加号前面的空格没用，不用管它。计算的时候呢，就是用加的方式把两个数字加起来，这一点计算机比人强，呵呵，告诉他怎么加永远不会错的。而且理解起加减乘除先天有优势。 

```
public class Add implements MathInterface {  
    static String plusPattern = BLANK + MathNumber.numberPattern + BLANK  
            + "[+]{1}" + BLANK + MathNumber.numberPattern + BLANK;  
    static Pattern pattern = Pattern.compile(plusPattern);  
    static Pattern plus = Pattern.compile(BLANK + "\\+");  
   
    @Override  
    public Matcher match(String string) {  
        return pattern.matcher(string);  
    }  
   
    @Override  
    public int priority() {  
        return 1;  
    }  
   
    @Override  
    public String eval(String expression) {  
        Matcher a = MathNumber.pattern.matcher(expression);  
        if (a.find()) {  
            expression = expression.substring(a.end());  
        }  
        Matcher p = plus.matcher(expression);  
        if (p.find()) {  
            expression = expression.substring(p.end());  
        }  
        Matcher b = MathNumber.pattern.matcher(expression);  
        if (b.find()) {  
   
        }  
        return new BigDecimal(a.group()).add(new BigDecimal(b.group()))  
                .toString();  
    }  
   
}  
```

接下来是括号，括号的优先级是最大啦，只要有它就应该先计算。当然，要先计算最内层的括号中的内容。括号中的内容，计算的时候，可以先拉出来，不用管外面的内容，计算好了，放回去就可以了。 

```
public class Bracket implements MathInterface {  
   
    static String bracketPattern = BLANK + "[(]{1}[^(]*?[)]" + BLANK;  
    static Pattern pattern = Pattern.compile(bracketPattern);  
   
    @Override  
    public Matcher match(String string) {  
        return pattern.matcher(string);  
    }  
   
    @Override  
    public int priority() {  
        return Integer.MAX_VALUE;  
    }  
   
    @Override  
    public String eval(String expression) {  
        expression = expression.trim();  
        return MathEvaluation.eval(expression.substring(1,  
                expression.length() - 1));  
    }  
   
}  
```

到目前为止，我们的程序“宝宝”已经学会数学计算了，出个题让伊试试。 

```
public static void main(String[] args) {  
String string = "1+2^(4/2)+5%2";  
System.out.println("结果是 :" + MathEvaluation.eval(string));  
}  
```

程序宝宝的做题过程如下：

```
求解算式：1+2^(4/2)+5%2  
发现算式：(4/2)  
求解算式：4/2  
发现算式：4/2  
4/2计算结果为：2.00,代回原式  
(4/2)计算结果为：2.00,代回原式  
求解算式：1+2^2.00+5%2  
发现算式：2^2.00  
2^2.00计算结果为：4,代回原式  
求解算式：1+4+5%2  
发现算式：5%2  
5%2计算结果为：1,代回原式  
求解算式：1+4+1  
发现算式：1+4  
1+4计算结果为：5,代回原式  
求解算式：5+1  
发现算式：5+1  
5+1计算结果为：6,代回原式  
结果是 :6  
```

呵呵，程序宝宝的做题过程和人的做题过程非常一致，而且程序实现也非常简单易懂。神马编译原理，神马中缀表达式都用不上。(执行效率与其它算法比较不一定高，仅用于验证通过规则让程序的处理能力增强，由于没有进行深入测试，正则表达式和程序逻辑是否写得严密没有经过深入验证)  

其实程序虽然很简单，但是，实际上已经是一个简单的规则引擎的雏形。  
首先，他加载了许多的业务处理规则，加，减，乘，除，插号，指数，余数等等。  

第二，他的业务规则是可以不断进行扩展的。  
第三，只要给出事实，最后，他通过规则的不断应用，最后会导出结果，要么是正确的结果，要么说给出的事实是错误的。  

需要源码的童鞋请到GIT上直接获取代码。

git地址：<http://git.oschina.net/tinyframework/mathexp.git>