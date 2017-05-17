异常：对于Java应用来说，异常的捕获和处理是非常消耗资源的，如果程序高频率的进行异常处理，则整体性能便会有明显下降。
数据库：海量数据的读写操作可能是相当费时的，而应用程序可能需要等待数据库操作完成或者返回请求的结果集，那么缓慢的同步操作会成为系统性能的瓶颈。
锁竞争：对高并发程序来说，如果存在激烈的锁竞争，对性能有极大的打击。锁竞争会明显增加线程上下文切换的开销，而且这些开销都是与应用需求无关的系统开销，白白占用cpu资源，带不来任何好处。

单例模式：
1，对于频繁使用的对象，可以省略创建对象所花费的时间，这对于那些重量级对象而言，是非常可观的系统开销
2，由于new操作的次数减少，因而对系统内存的使用频率也会降低，这将减轻压力，缩短GC时间。

静态内部类实现方式:
public class StaticSingleton {

	private StaticSingleton(){
		System.out.println("StaticSingleton is create");
	}
	private static class SingletonHolder {
		private static StaticSingleton instance = new StaticSingleton();
	}

	public static StaticSingleton getInstance() {
		return SingletonHolder.instance;
	}
	public static void createString(){
		System.out.println("createString in Singleton");
	}
}
单例模式使用内部类来维护单例的实例，当StaticSingleton被加载时，其内部类并不会被初始化，故可以确保当StaticSingleton类被载入JVM时，不会初始化单例类，而当getInstance()方法被调用时，才会加载SingletonHolder，从而初始化instance。同事，由于实例的建立在类加载时完成，故天生对线程友好，getInstance()方法也不需要使用同步关键字。
使用内部类实现单例模式，即可以做到延迟加载，也不必使用同步关键字，是一种比较完善的实现。

可以通过反射机制，强行调用单例类的私有构造函数，生成多个单例。
Singleton s=Singleton.getInstance();
		Singleton s1=null;
		Class singletonClass = Singleton.class;
		Constructor cons;
		try {
			cons = singletonClass.getDeclaredConstructor(null);
			cons.setAccessible(true);
			s1 = (Singleton)cons.newInstance(null);
		}
    ......

序列化和反序列化可能会破坏单例。在实现Serializable接口后，重写readReslove()函数，就可以防止这种情况，这种情况下，即使经过反序列化，仍然保持了单例的特点。事实上，在实现了私有的readReslove方法后，readObject()以及形同虚设，它直接使用readSlove()替换了原本的返回值，从而在形式上构造了单例。

代理模式：
使用代理对象完成用户请求，屏蔽用户对真是对象的访问。在软件设计中，使用代理模式的意图很多，比如因为安全原因，需要屏蔽客户端直接访问真是对象；或者在远程调用中，需要使用代理类处理远程方法调用的技术细节(比如RMI)；也可能为了提升系统性能，对真实对象进行封装，从而达到延迟加载的目的。

角色               作用
主题接口      定义代理类和真实主题的公共对外方法，也是代理类代理真实主题的方法
真实主题      真正实现业务逻辑的类
代理类        用来代理和封装真实主题
Main         客户端，使用代理类和接口主体完成具体业务

代理模式实现延迟加载的方法及其意义：
假设某客户端软件，有根据用户请求去数据库查询数据的功能。在查询数据前，需要获得数据库连接，软件开启时，初始化系统的所有类，此时尝试获得数据库连接。当系统有大量的类似操作存在时，比如xml解析，所有这些操作的叠加，会使得系统的启动速度变得非常的缓慢，初始化这个代理类，而非真实的数据库查询类，而代理类什么都没有做，因此，他的构造是相当迅速的。在系统启动时，讲消耗资源最多的方法都使用代理模式分离，就可以加快系统的启动速度，减少用户的等待时间。而在用户真正做查询时，再由代理类单独去加载真是的数据库查询类，完成用户的请求。这个过程就是使用代理模式实现了延迟加载。
延迟加载的核心思想是，如果当前并没有使用这个组件，则不需要真正的初始化它，使用一个代理对象代替它的原有的位置，只要在真正使用的时候，才对它进行加载。使用代理模式的延迟加载是非常有意义的，首先它可以再时间轴上分散系统压力，尤其是在系统启动时，不必完成所有的初始化工作，从而加速启动时间；其次，对很多真实主题而言，在软件启动直到被关闭的整个过程中，可能更本不会被调用，初始化这些数据无疑是一种资源的浪费。

动态代理：
动态代理是指在运行时，动态生成代理类，即，代理类的字节码将在运行时生成并加载入当前的ClassLoader。与静态代理对比，动态代理有组多好处。首先，不需要为真实主题写一个形式上完全一样的类，假如主题接口中的方法很多，为每一个借口写一个代理类也是很烦人的事。如果接口有变动，则真实主题和代理类都要修改，不利于系统维护；其次，使用一些动态代理生成方法可以在运行时指定代理类的执行逻辑，从而提升系统的灵活性。
动态代理使用字节码动态生成加载技术，在运行时生成并加载类。
常用的动态代理，JDK自带的动态代理，CGLIB，Javassist，ASM库。
JDK的动态代理使用简单，内置在JDK中，因此不需要引入第三方jar，但相对功能较弱。
CGLIB和Javassit都是高级的字节码生成库，总体性能比JDK自带的动态代理好。
ASM时低级的字节码生成工具，使用ASM已经近乎在使用Java bytecode编程，对开发人员的要求最高。当然也是性能最好的一种动态代理生成工具。ASM使用是在复杂，而且性能没有数量级的提升。

JDK方式：
public class JdkDbQeuryHandler implements InvocationHandler {
	IDBQuery real=null;

	public Object invoke(Object proxy, Method method, Object[] args)
			throws Throwable {
		if(real==null)
			real=new DBQuery();
		return real.request();
	}
}
public static IDBQuery createJdkProxy(){
		IDBQuery jdkProxy = (IDBQuery) Proxy.newProxyInstance(ClassLoader.getSystemClassLoader(),  
	                new Class[] { IDBQuery.class }, new JdkDbQeuryHandler());  
	        return jdkProxy;  
	}

CGLIB方式：
public class CglibDbQueryInterceptor implements MethodInterceptor {
	IDBQuery real=null;
	public Object intercept(Object arg0, Method arg1, Object[] arg2,
			MethodProxy arg3) throws Throwable {
		if(real==null)
			real=new DBQuery();
		return real.request();
	}
}

public static IDBQuery createCglibProxy(){
      Enhancer enhancer = new Enhancer();  
      enhancer.setCallback(new CglibDbQueryInterceptor());  
      enhancer.setInterfaces(new Class[] { IDBQuery.class });  
      IDBQuery cglibProxy = (IDBQuery) enhancer.create();  
      return cglibProxy;  
}
实现动态代理的基本步骤：
1，根据指定的回调类生成Class字节码
2，通过defineClass()将字节码定义为类
3，使用反射机制生成该类的实例

HIbernate中代理模式的使用：
当Hibernate加载实体bean时，并不会一次性将数据库所有的数据都装载。默认情况下，它会采取延迟加载的机制，以提高系统的性能。Hibernate中的延迟加载主要有两种：一是属性的延迟加载，二是关联表的延时加载。
User u=(User)HibernateSessionFactory.getSession().load(User.class, 1);
System.out.println("Class Name:"+u.getClass().getName());
System.out.println("Super Class Name:"+u.getClass().getSuperclass().getName());
Class[] ins=u.getClass().getInterfaces();
for(Class cls:ins){
	System.out.println("interface:"+cls.getName());
}
System.out.println(u.getName());
以上代码中，在session.load()方法后，首先输出了User的类名，它的超类，User实现的接口，最后输出调用User的getName()方法取得数据库连接。
session的载入类并不是之前的User类，而是一个CGLIB的Enhancer类生成的动态类。该类的父类才是应用程序定义的User类。
此外，它实现了HibernateProxy接口。由此可见，Hibernate使用一个动态代理子类代替用户定义的类。这样，在载入对象时，就不必初始化对象的所有信息，通过代理，拦截原有的getter方法，可以在真正使用对象数据时采取数据库加载实际的数据，从而提升系统性能。
也就是，在getName()b被调用之前，Hibernate从未输出国一句SQL，这表示，User对象被加载时，根本没有访问数据库，而在getName()方法被调用时，才真正完成了数据库操作。

享元模式： 以提高系统性能为目的。主要目的时复用大对象，以节省内存空间和对象创建时间
如果在一个系统中存在多个相同的对象，那么只需要共享一份对象的拷贝，而不必为每一次使用都创建新的对象。在享元模式中，由于需要构造和维护这些可以共享的对象，因此，常常会实现一个工厂类，用于维护和创建对象。
享元模式对性能的组要帮助有两点：
1，可以节省重复创建对象的开销，因为被享元模式维护的相同的对象只会被创建一次，当创建对象比较耗时时，可以节省大量时间
2，由于创建对象的数量少，所以对系统内存的需求也减少，这将使得GC的压力也相应的降低，进而使得系统拥有一个更健康的内存结构和更快的反应速度。

角色              作用
享元工厂        用以创建具体享元类，维护相同的享元对象，它保证相同的享元对象可以直接被系统共享，
               即，其内部使用了类似与单例模式的算法，当请求对象以及存在时，直接返回对象，不存在时，再创建对象。
抽象享元        定义需要共享的对象的业务接口，享元类被创建出来总是为了实现某些特定的业务逻辑，而抽象享元便定义
               这些逻辑的语义行为
具体享元类      实现抽象享元的接口，完成某一具体逻辑
Man           使用享元模式的组件，通过享元得到享元对象
享元工厂是享元模式的核心，它需要确保系统可以共享相同的对象。一般情况下，享元工厂会维护一个对象列表，当任何组件尝试获取享元类时，如果请求的享元类已经被创建，则直接返回已有的享元类；若没有，则创建一个新的享元对象，并将它加入到维护列表中。
