--- 
layout: post
title: "PanMX: Exposing MBeans the easy way"
---
PanMX removes the pain from developing MBeans. It was a toolkit I developed years ago to make things less painful. I have been cleaning up my subversion repositories and I figured it may be useful for others. So here it is released to the public at large.

The user can use Java 1.5 Annotations to define the MBean interface or allow the toolkit to derive the MBean interface using reflection.

The toolkit allows the user to define [ModelMBeans](http://java.sun.com/j2se/1.5.0/docs/api/javax/management/modelmbean/ModelMBean.html) via the [ModelMBeanFactory](http://realityforge.org/static/PanMX/api/panmx/model/ModelMBeanFactory.html) or RMXBeans which are very similar [MXBeans](http://java.sun.com/j2se/1.5.0/docs/api/java/lang/management/ManagementFactory.html#MXBean) via the [RMXBeanFactory](http://realityforge.org/static/PanMX/api/panmx/rmx/RMXBeanFactory.html) .

### Annotating a Class

An example of how to annotate a Java Class.

{% highlight java %}
@MBean(description = “Component to measure and clean out toxins”)
public class Liver
{
private float m\_toxicity;

@MxAttribute()
public float getToxicity()
{
return m\_toxicity;
}

@MxAttribute(description = “Level of toxicity”)
public void setToxicity(final float toxicity)
{
m\_toxicity = toxicity;
}

@MxOperation(description = “Clean out the toxins”)
public void clean()
{
m\_toxicity = 0;
}
}
{% endhighlight %}

### Exposing an MBean on the ‘Server’

To expose an instance of the Liver class as a ModelMBean the user could add the following code;

{% highlight java %}
final Liver liver = new Liver();
final ObjectName liver = new ObjectName(“body:part=Liver”);
final ModelMBean mBean =
ModelMBeanFactory.createAnnotatedModelMBean( liver );
final MBeanServer mBeanServer =
ManagementFactory.getPlatformMBeanServer();
mBeanServer.registerMBean( mBean, objectName );
{% endhighlight %}

To expose the object as an RMXBean the user could use the following code;

{% highlight java %}
final Liver liver = new Liver();
final ObjectName liver = new ObjectName(“body:part=Liver”);
final Object mBean =
RMXBeanFactory.createAnnotatedRMXBean( liver );
final MBeanServer mBeanServer =
ManagementFactory.getPlatformMBeanServer();
mBeanServer.registerMBean( mBean, objectName );
{% endhighlight %}

### Accessing an MBean on the ‘Client’

The user can access the MBeans using the standard mechanisms. Additionally if the user decides to use RMXBeans and has defined the management interface using a java interface then the object can be dynamically proxied. This means that the user can interact with the object as if it was a local object.

The following code example demonstrates accessing a RMXBean as a dynamic proxy.

{% highlight java %}
public interface LiverRMXBean
{
@MxAttribute()
float getToxicity();

@MxAttribute(description = “Level of toxicity”)
void setToxicity(float toxicity);

@MxOperation(description = “Clean out the toxins”)
void clean();
}

@MBean(description = “Component to measure and clean out toxins”,
interfaces = {LiverMBean.class})
public class Liver
implements LiverMBean
{
private float m\_toxicity;

public float getToxicity()
{
return m\_toxicity;
}

public void setToxicity(final float toxicity)
{
m\_toxicity = toxicity;
}

public void clean()
{
m\_toxicity = 0;
}
}

…
// Registering the Liver MBean
final Liver liver = new Liver();
final ObjectName liver = new ObjectName(“body:part=Liver”);
final Object mBean =
RMXBeanFactory.createAnnotatedRMXBean( liver );
final MBeanServer mBeanServer =
ManagementFactory.getPlatformMBeanServer();
mBeanServer.registerMBean( mBean, objectName );
…

…
// Accessing and using the Liver MBean
final MBeanServerConnection connection = …;
final ObjectName liver = new ObjectName(“body:part=Liver”);
final LiverRMXBean liver = (LiverRMXBean)
RMXBeanFactory.newProxyInstance( liver );

System.out.println( “Current Toxicity Level: ” + liver.getToxicity() );
liver.clean();
System.out.println( “Toxicity Level after cleaning: ” + liver.getToxicity() )
…
{% endhighlight %}

### Files

To get your greasy mits on PanMX visit the [GitHub project page](http://github.com/realityforge/panmx)
