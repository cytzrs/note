1,字符串优化处理
Java中的String类主要由3部分组成：char数组，偏移量和String的长度。char数组表示String的内容，它是String对象所表示字符串的超集。String的真实内容还需要有其他偏移量和长度在这个char数组中进行定位和截取。
String str1 = "abc";
String str2 = "abc";
String str3 = new String("abc");

System.out.println(str1 == str2);          true
System.out.println(str1 == str3);          false
System.out.println(str1 == str3.intern()); true
str1和str2引用了相同的地址，但是str3却重新开辟了一块内存空间，但即便如此，str3在常量池中的位置和str1是一样的，也就是说虽然str3单独占用了堆空间，但是它所指向的实体和str1完全一样。inter()方法返回String对象在常量池中的引用。

使用final定义，有助于虚拟机关联所有的final方法，从而提高系统效率。

tatic class HugeStr {
	private String str = new String(new char[100000]);
	public String getSubString(int begin, int end) {
		return str.substring(begin, end);
	}
}

static class ImprovedHugeStr {
	private String str = new String(new char[100000]);
	public String getSubString(int begin, int end) {
		return new String(str.substring(begin, end));
	}
}
ImprovedHugeStr能够很好的工作的关键是因为它使用没有内存泄露的String构造函数重新生成了String对象，使得由subString()方法返回的，存在内存泄漏问题的String对象失去所有的强引用，从而被垃圾回收期识别为垃圾对象进行回收，保证了系统内存的稳定。

2，字符串分割和查找
public String[] split(String regax);
String.split()方法使用简单，功能强大，但是，在性能敏感的系统中频繁使用这个方法是不可取的。

public StringTokenizer(String str, String delim)
其中str参数是要分割处理的字符串，delim时分隔符号。当一个StringTokenizer对象生成后，通过它的nextToken()方法可以得到下一个分割的字符串。通过hasMoreTokens()方法可以知道是否有更多的子字符串需要处理。

更优化的字符串分割方式
使用indexOf()和subString()完成一个优化的字符串分割算法。
String tmp=orgStr;
for(int i=0;i<10000;i++){
 while(true){
	String splitStr=null;
  int j=tmp.indexOf(';');
  if(j<0)break;
  splitStr=tmp.substring(0,j);
  tmp=tmp.substring(j+1);
 }
 tmp=orgStr;
}
综合来说，split()方法功能强大，但是效率最差；StringTokenizer性能优于split()方法，因此，在能够使用StringTokenizer的模块中，就没有必要使用split();而由自己实现的分割算法性能最好。

3，StringBuffer和StringBuilder
由于String对象是不可变对象，因此在需要对字符串进行修改操作时，String对象总是会生成新的对象，所以性能相对较差。
3.1，String常量的累加操作
String result = "String" + "and" + "String" + "append";
首先，"string"和"and"生成"Stringand"对象，再依次生成"StringandString"和"StringandStringappend"对象

StringBuilder result = new StringBuilder();
result.append("String");
result.append("and");
result.append("String");
result.append("append");
只生成一个实例result，并通过StringBuilder的append()方法向其中追加字符串。

然而运行的结果是，前者运行0s，后者运行15ms。
分析：对于常量字符串的累加，Java在编译时就做了充分的优化。对于这些在编译时便能确定取值的字符串操作，在编译时就进行了计算，因此在运行时这段代码并没有如想象中那样产生大量的String实例。对于静态字符串的连接操作，Java在编译时会进行彻底的优化，将多个连接操作的字符串在编译时合成一个单独的长字符串。
3.2，String变量的累加操作
对于变量字符串的累加，Java也做了相应的优化操作，使用了StringBuilder对象来实现字符串的累加。所以，这段代码的性能和直接使用StringBuilder类的性能几乎一样。
Java在编译时，就会对字符串进行一定的优化。因此，一些看起来会很慢的代码，可能实际上并不慢。
3.3，构建超大的String对象
for(int i = 0; i < 10000; i++) {
	str = str + i;
} 被Java编译器编译成:
for(int i = 0; i < CIRCLE; i++) {
	str = (new StringBuilder(String.valueOf(str))).append(i).toString();
}
以上反编译代码显示，虽然String的加法运行被编译成StringBuilder的实现，但在这种情况下，编译器并没有做出足够聪明的判断，每次循环都生成了新的StringBuilder实例从而大大降低了系统性能。
3.4，StringBuilder和StringBuffer的选择
StringBuilder，StringBuffer都实现了AbstractStringBuilder抽象类，拥有几乎相同的对外接口。两者最大的不同在于StringBuffer对几乎所有的方法都做了同步，而StringBuilder并没有做任何同步。因此，StringBuilder的效率也好于StringBuffer。StringBuilder无法保证线程安全，因此在多线程系统中不能使用。
在非同步的StringBuilder拥有更多的效率。在无需考虑线程安全的情况下可以使用性能相对较好的StringBuilder，但若系统有线程安全要求，只能选择StringBuffer。
StringBuilder/StringBuffer，在不指定容量参数时，默认是16个字节。在追加字符串时，如果需要容量超过实际char数组长度，则需要进行扩容。扩容策略是将原有的容量大小翻倍，以新的容量申请内存大小，建立新的char数组，然后将原数组中的内容复制到这个新的数组中。因此，对于大对象的扩容会涉及大量的内存复制操作。所以，如果能够预先评估StringBuilder的大小，将能够有效地节省这些操作，从而提高系统的性能。

Java核心数据结构
List接口：
List的三种实现：ArrayList，Vector和LinkedList
ArrayList和Vector使用了数组实现，也可以认为ArrayList或者Vector封装了对内部数组的操作。比如向数组中添加，删除，插入新的元素或者数组的扩展和重定义。
ArrayList和Vector几乎使用了相同的算法，它们的唯一区别可以认为是对多线程的支持。ArrayList没有对任何一个方法做线程同步，因此不是线程安全的。Vector中绝大部分方法都做了线程同步，是一种线程安全的实现。因此，ArrayList和Vector的性能相差无几，从理论上说，没有实现线程同步的ArrayList要稍好于Vector，但实际表现并不是非常明显。
LinkedList使用了循环双向链表数据结构，在JDK的实现中，无论LinkedList是否为空，链表内都有一个header表项，它即表示链表的开始，也表示链表的结尾。表项header的后驱表项便是链表便是链表中第一个元素，表项header的前驱表项便是链表中最后一个元素。

Hashtable与HashMap区别：
1，Hashtable大部分方法做了同步，而HashMap没有，因此HashMap不是线程安全的。
2，Hashtable不允许key或者value使用null值，而HashMap可以
3，它们对key的hash算法和hash值到内存索引的映射算法不同

主要的区别有：线程安全性，同步(synchronization)，以及速度。

HashMap几乎可以等价于Hashtable，除了HashMap是非synchronized的，并可以接受null(HashMap可以接受为null的键值(key)和值(value)，而Hashtable则不行)。
HashMap是非synchronized，而Hashtable是synchronized，这意味着Hashtable是线程安全的，多个线程可以共享一个Hashtable；而如果没有正确的同步的话，多个线程是不能共享HashMap的。Java 5提供了ConcurrentHashMap，它是HashTable的替代，比HashTable的扩展性更好。
另一个区别是HashMap的迭代器(Iterator)是fail-fast迭代器，而Hashtable的enumerator迭代器不是fail-fast的。所以当有其它线程改变了HashMap的结构（增加或者移除元素），将会抛出ConcurrentModificationException，但迭代器本身的remove()方法移除元素则不会抛出ConcurrentModificationException异常。但这并不是一个一定发生的行为，要看JVM。这条同样也是Enumeration和Iterator的区别。
由于Hashtable是线程安全的也是synchronized，所以在单线程环境下它比HashMap要慢。如果你不需要同步，只需要单一线程，那么使用HashMap性能要好过Hashtable。
HashMap不能保证随着时间的推移Map中的元素次序是不变的。

TreeMap提供的其他的有关排序的接口:
SortedMap<K, V> subMap(K fromKey, K toKey);  
SortedMap<K, V> headMap(K toKey);
SortedMap<K, V> tailMap(K fromKey);
K firstKey();
K lastKey();
