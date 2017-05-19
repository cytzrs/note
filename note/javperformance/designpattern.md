装饰着模式：可以有效分离性能组件和功能组件，从而提升模块的可维护性并增加模块的复用性
装饰着模式是一个设计非常巧妙地结构，它可以动态的添加对下功能。在基本的设计原则中，有一条重要的设计准则叫合成/聚合复用原则。根据该原则的思想，代码复用应该尽可能使用委托，而不是使用继承。因为继承是一种紧密耦合，任何父类的改动都会影响其子类，不利于系统维护。而委托则是松散结构，只要接口不变，委托类的改动并不会影响其上层对象。
DataOutputStream dout=new DataOutputStream(new BufferedOutputStream(new FileOutputStream("C:\\a.txt")));
//DataOutputStream dout=new DataOutputStream(new FileOutputStream("C:\\a.txt"));
long begin=System.currentTimeMillis();
for(int i=0;i<100000;i++)
	dout.writeLong(i);
System.out.println("spend:"+(System.currentTimeMillis()-begin));
OutputStream为核心的装饰着模式实现。其中FileOutputStream为系统的核心类，它实现了向文件写入数据。使用DaaOutputStream可以再FileOutputStream的基础上，增加对多种类型的文件的写操作支持，而BufferedOutPutStream装饰器，可以对FileOutputStrea增加缓冲功能，优化I/O性能。以FufferedOutputStream为代表的性能组件，是将性能模块和功能模块分离的一种典型实现。

JDK中OutputStream和InputStream类族的实现是装饰者模式的典型应用。通过嵌套的方式不断地将对象聚合起来，最终形成一个超级对象，并使之拥有所有权相关子对象的功能。

观察者模式：
当一个对象的行为依赖于另一个对象的状态时，观察者模式就相当的有用。若不使用观察者模式提供的通用结构，而需要实现其类似的功能，则只能在另一个线程中不停地监听对象所依赖的状态。在一个复杂系统中，可能会因此开启很多线程来实现这一功能，这将是系统的性能产生额外的负担。观察者模式的意义就在于此，它可以在单线程中，是某一对象，及时得知自身所依赖的状态的变化。
ISubject是被观察对象，它可以增加或者删除观察者。IOberver是观察者，它依赖于ISubject的状态变化。当ISubject状态发生改变时，会通过inform()方法通知观察者。
观察者模式可以用于事件监听，通知发布等场合。可以确保观察者在不使用轮询监控的情况下，及时收到相关消息和事件。

public interface ISubject{  
    void attach(IObserver observer);	//添加观察者  
    void detach(IObserver observer);	//删除观察者  
    void inform();					//通知所有观察者  
}  
public interface IObserver{  
    void update(Event evt);				//更新观察者  
}
public class ConcreteSubject implements ISubject{
	Vector<IObserver> observers=new Vector<IObserver>();
    public void attach(IObserver observer){  
    	observers.addElement(observer);  
    }  
    public void detach(IObserver observer){  
    	observers.removeElement(observer);  
    }  
    public void inform(){
    	Event evt=new Event();
    	for(IObserver ob:observers){
    		ob.update(evt);
    	}
    }  
}
public class ConcreteObserver implements IObserver{  
    public void update(Event evt){  
    	System.out.println("obserer receives information");
    }  
}
public class Event {

}

在java.util.Observable类中，已经实现了主要的功能，如增加观察者，删除观察者和通知观察者，开发人员可以直接通过继承Observable使用这些功能。java.util.Observer接口时观察者接口，他的update()方法会在java.util.Observable的notifyObservers()方法中被回调，以获取最新的状态变化。通常在观察者模式中Observer接口总是应用程序的核心扩展对象，具体业务总是会封装在update()方法中。


Value Object模式：
在J2EE软件开发中，通常会对系统模块进行分层。展示层主要负责数据的展示，定义数据库的UI组织模式；业务逻辑层负责具体的业务逻辑处理；持久层通常指数据库以及相关操作。在一个大型系统中，这些层次很有可能被分离，部署在不同的服务器上。而在两个层次之间，可能通过远程过程调用RMI等方式进行通信。
使用Value Object模式可以有效减少网络交互次数，提高远程调用方法的性能，也可使系统接口具有更好的可维护性。
Value Object模式提倡将一个对象的各个属性进行封装，将封装后的对象在网络中传输，从而使系统拥有更好的交互模型，并且减少网络通信数据，从而提高系统性能。Value Object必须是一个可串行化的对象。

业务代理模式：
Value Object模式是将远程调用的传递数据封装在一个串行化的对象中进行传输，而业务代理模式则是将一组由远程方法调用构成的业务流程，封装在一个位于展示层的代理类中。
业务代理对象负责和远程服务器通信。业务代理模式将一些业务流程封装在前台系统，为系统性能优化提供了基础平台，在业务代理中，不仅可以服用业务流程，还可以视情况为展示层组价提供缓存等功能，从而减少远程方法调用此书，降低系统压力。
