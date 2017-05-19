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
