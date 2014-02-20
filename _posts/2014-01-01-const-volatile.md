---
layout: post
title: "const volatile修饰的变量"
description: ""
category: 
tags: [苏州大学]
---
{% include JB/setup %}
# 1 const volatile修饰的意义 
　　volatile的本意是“易变的” 因为访问寄存器要比访问内存单元快的多,所以编译器一般都会作减少存取内存的优化，但有可能会读脏数据。当要求使用volatile声明变量值的时候，系统总是重新从它所在的内存读取数据，即使它前面的指令刚刚从该处读取过数据。精确地说就是，遇到这个关键字声明的变量，编译器对访问该变量的代码就不再进行优化，从而可以提供对特殊地址的稳定访问；如果不使用valatile，则编译器将对所声明的语句进行优化。（简洁的说就是：volatile关键词影响编译器编译的结果，用volatile声明的变量表示该变量随时可能发生变化，与该变量有关的运算，不要进行编译优化，以免出错）  
　　const和volatile 也一样，所谓的const，只是编译器保证在C的“源代码”里面，没有对该变量进行修改的地方，而实际运行的时候则不是编译器所能管的了。  
　　同样，volatile的所谓“可能被修改”，是指“在运行期间”可能被修改。也就是告诉编译器，这个变量不是“只”会被这些C的“源代码”所操纵，其它地方也有操纵它们的地方。所以，C编译器就不能随便对它进行优化了。  
　　const    -->该变量为常量,不能在此程序中更改  
　　volotile -->该变量为一个共享变量,也就是说会有除了本程序之外的其他途径对其值进行更改,如多线程,或是硬件，其他的运行程序.    
　　const volatile表示该变量既不能被修改，又不能被优化到寄存器，即又是可能会被其他编译器不知道的方式修改的。比如一个`实时时钟`，我们不希望被程序做修改，所以要声明为const，但其他的线程、中断等（可能来自于库）又要修改此时钟的值，编译器不能把它作为const常量优化到寄存器，所以又要声明为volatile。再举个例子，`只读的状态寄存器`，它是volatile，因为它可能被意想不到地改变。它是const，因为程序不应该试图去修改它。


# 2 const的修饰作用
　　对于所有的关键字，我们应该关注量方面：`修饰域`（即这个关键字修饰那块范围）和`修饰方向`（即从哪里开始修饰）。对于c类似的语言(c,c++,java),关键字的修饰方向都是从向右的，即关键字不会修饰它左边的东西。之前学编译原理的时候有讲过，但是具体细节不是很记得了，只是记得和语法分析有关，我们现在编译器采用的基本都是LR(k)进行语法分析，L 指自左至右扫描输入符号串，R 指构造一个最右推导的逆过程（最左归约），k 指在作出分析决定前要向前看的输入符号个数，通常为1。详细内容请参考[wiki](http://en.wikipedia.org/wiki/LR_parser)。下面介绍的是在不同场合const的修饰作用。

### 2.1 修饰变量 ###
　　举例：`const char a;`和`char const a;`  
　　C++标准中规定了类型前或者变量前是等价的，即这两者的意义相同，都是代表a是一个只读变量。虽然是只读变量，但是并不代表不能够被修改。

### 2.2 与指针一起使用 ###
　　<<C++ Primer>>中介绍的方法我觉得我蛮好记的，就是将*号读作pointer to，然后从右向左读表达式。


- `const char *p` =>读作p is a pointer to const char    
　　表明p是一个常量指针，指向const char，也就是p可以改变，但是p指向的值不可以改变。

- `char const *p` =>同上面一样,读作p is a pointer to const char     
　　C++标准中规定了类型前或者变量前是等价的，同上面一样，读作p is a pointer to const char ，表明p是一个常量指针，指向const char，也就是p可以改变，但是p指向的值不可以改变。  

- `char * const p` =>读作p is a const pointer to char  
　　表明p是一个指针常量，指向char类型，也就是p不可以改变，但是p指向的值可以改变。

### 2.3 修饰函数参数 ###
　　举例：`void foo(const char* a);`和`void foo(char * const a);`  
　　这个直接参考第二点就可以了，前者表示a指向的数据不能改，后者表示a不能改。	
　　

# 3 修改const常量
　　既然const修饰的变量的是只读的，那么很自然的想到能不能取得变量的地址然后直接修改地址处的值来达到修改const变量的目的。写个简单的程序验证下可不可以：
{% highlight c++ %}
 #include <stdio.h>
 int main()
 {
	const int a = 10;
    int *a_ptr=(int*)&a;
    *a_ptr=1000;
	printf("a = %d\n",a ); 
    printf("*a_ptr = %d\n",*a_ptr);

}
{% endhighlight %}  
运行环境：`ubuntu 12.04.3 gcc `  
这个程序运行出来结果如下：
![](http://d.pcs.baidu.com/thumbnail/f4b0a3c6dfb1328abd9f0252c699eb0b?fid=859179042-250528-88872591&time=1392863793&rt=pr&sign=FDTAER-DCb740ccc5511e5e8fedcff06b081203-fNJdHdLF4oP4OBMZrBU2gpT2VXs%3D&expires=8h&prisign=RK9dhfZlTqV5TuwkO5ihMQzlM241kT2YfffnCZFTaEPwOxHv/XxtwRXLxDSXMBba1Ms9seOiqT9/QffwI8K2Baw0mmLABRQNl51b/oS8+InqoadADmwcyvUvQUkAvl8j879BObtgHePZ09BZiFo3CF7dwGcK/xIzausW8RTta6nycgI5A4W/Ju5XK/lbx40b7xvlT+8yG2JtdgCRiVnR1D9eX47fiExbpbz9eZn02Sc=&r=147735928&size=c850_u580&quality=100)

和我们猜想的一致。由此看出，const关键字只是在显式的代码层面帮我们检查是否修改了只读变量，并不能保证在代码执行过程中保证const修饰的变量没有被修改，也验证了第一点中const volatile修饰变量并不矛盾。