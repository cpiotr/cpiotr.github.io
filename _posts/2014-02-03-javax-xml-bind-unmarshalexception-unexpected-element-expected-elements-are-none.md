---
ID: 183
post_title: 'javax.xml.bind.UnmarshalException: unexpected element. Expected elements are (none)'
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2014/02/javax-xml-bind-unmarshalexception-unexpected-element-expected-elements-are-none/
published: true
post_date: 2014-02-03 17:11:47
---
While providing a stub for SOAP web service endpoint a need occured to return real-life data from production environment to track a <a href="http://en.wikipedia.org/wiki/Heisenbug" target="_blank">heisenbug</a>.
Such data was available in form of rather spacious XML element. It wasn't a complete document, but a non-root part of SOAP payload.
JAXB Model was generated from XML Schema, however when trying to naively deserialize data aformentioned error popped up: <code>javax.xml.bind.UnmarshalException: unexpected element (uri:"", local:"Yadayada"). Expected elements are (none)</code>.

[sourcecode lang="java"]
String xml =
    &quot;&lt;?xml version=\&quot;1.0\&quot; encoding=\&quot;UTF-8\&quot; standalone=\&quot;yes\&quot;?&gt;&quot; +
    &quot;&lt;Player&gt;&quot; +
        &quot;&lt;firstName&gt;Stiopan&lt;/firstName&gt;&quot; +
        &quot;&lt;lastName&gt;Lafayete&lt;/lastName&gt;&quot; +
    &quot;&lt;/Player&gt;&quot;;

JAXBContext jaxbContext = null;
try {
    InputStream inputStream = new ByteArrayInputStream(xml.getBytes());
   
    jaxbContext = JAXBContext.newInstance(Player.class);
    Unmarshaller unmarshaller = jaxbContext.createUnmarshaller();
    Player apprentice = (Player) unmarshaller.unmarshal(inputStream);
   
    ...
} catch (Exception e) {
    e.printStackTrace();
}
[/sourcecode]

The key point to remember is that complex types from XML schema have their metadata mapped to annotated methods in <code>ObjectFactory</code>. Instantiating <code>JaxbContext </code>on non-root class does not provide sufficient information for deserialization.
Instead, package name or <code>ObjectFactory </code>object must be used to fulfill the requirement.

[sourcecode lang="java"]
public static &lt;T&gt; String marshall(T entity) throws JAXBException {
    Class&lt;T&gt; entityClass = (Class&lt;T&gt;) entity.getClass();
   
    // At this point ObjectFactory is required within provided package
    JAXBContext jaxbContext = JAXBContext.newInstance(entityClass.getPackage().getName());
   
    Marshaller m = jaxbContext.createMarshaller();
   
    JAXBElement&lt;T&gt; element = new JAXBElement&lt;T&gt;(new QName(entityClass.getSimpleName()), entityClass, entity);
   
    StringWriter sw = new StringWriter();
    m.marshal(element, sw);
   
    return sw.toString();
}

public static &lt;T&gt; T unmarshall(InputStream is, Class&lt;T&gt; entityClass) throws JAXBException {
    // At this point ObjectFactory is required within provided package
    JAXBContext jaxbContext = JAXBContext.newInstance(entityClass.getPackage().getName());
   
    Unmarshaller jaxb = jaxbContext.createUnmarshaller();
    JAXBElement&lt;T&gt; entityElement = (JAXBElement&lt;T&gt;) jaxb.unmarshal(is);
   
    T entity = entityElement.getValue();
    return entity;
}

public static void main(String[] args) {
    String xml =
        &quot;&lt;?xml version=\&quot;1.0\&quot; encoding=\&quot;UTF-8\&quot; standalone=\&quot;yes\&quot;?&gt;&quot; +
        &quot;&lt;Player&gt;&quot; +
            &quot;&lt;firstName&gt;Stiopan&lt;/firstName&gt;&quot; +
            &quot;&lt;lastName&gt;Lafayete&lt;/lastName&gt;&quot; +
        &quot;&lt;/Player&gt;&quot;;
   
    Player p = new Player();
    p.setFirstName(&quot;Joe&quot;);
    p.setLastName(&quot;McQeerk&quot;);
    try {
        InputStream inputStream = new ByteArrayInputStream(xml.getBytes());
       
        System.out.println(marshall(p));
       
        Player player = unmarshall(inputStream, Player.class);
        inputStream.close();
       
        System.out.println(String.format(&quot;\nPlayer name: %s, %s&quot;, player.getLastName(), player.getFirstName()));
    } catch (Exception e) {
        e.printStackTrace();
    }
}
[/sourcecode]

For code completeness, the model information is provided below.

[sourcecode lang="java"]
package pl.ciruk.jaxbtest.entity;

import javax.xml.bind.annotation.XmlAccessType;
import javax.xml.bind.annotation.XmlAccessorType;
import javax.xml.bind.annotation.XmlElement;
import javax.xml.bind.annotation.XmlType;

@XmlAccessorType(XmlAccessType.FIELD)
@XmlType(name = &quot;Player&quot;, namespace = &quot;http://entity.jaxbtest.ciruk.pl/&quot;, propOrder = {
    &quot;firstName&quot;, &quot;lastName&quot;
})
public class Player {
    @XmlElement(name = &quot;firstName&quot;, nillable = true)
    private String firstName;

    @XmlElement(name = &quot;lastName&quot;, nillable = true)
    private String lastName;

    public String getFirstName() {
        return firstName;
    }

    public void setFirstName(String firstName) {
        this.firstName = firstName;
    }

    public String getLastName() {
        return lastName;
    }

    public void setLastName(String lastName) {
        this.lastName = lastName;
    }
}
[/sourcecode]

[sourcecode lang="java"]
package pl.ciruk.jaxbtest.entity;

import java.util.ArrayList;
import java.util.List;

import javax.xml.bind.annotation.XmlAccessType;
import javax.xml.bind.annotation.XmlAccessorType;
import javax.xml.bind.annotation.XmlElement;
import javax.xml.bind.annotation.XmlRootElement;
import javax.xml.bind.annotation.XmlType;

@XmlAccessorType(XmlAccessType.FIELD)
@XmlRootElement(name = &quot;Team&quot;)
@XmlType(name = &quot;Team&quot;, namespace = &quot;http://entity.jaxbtest.ciruk.pl/&quot;, propOrder = {
    &quot;players&quot;
})
public class Team {
   
    @XmlElement(name = &quot;players&quot;, nillable = true)
    private List&lt;Player&gt; players;

    public void setPlayers(List&lt;Player&gt; players) {
        this.players = players;
    }

    public List&lt;Player&gt; getPlayers() {
        if (players == null) {
            players = new ArrayList&lt;Player&gt;();
        }
        return players;
    }
}
[/sourcecode]

[sourcecode lang="java"]
package pl.ciruk.jaxbtest.entity;

import javax.xml.bind.JAXBElement;
import javax.xml.bind.annotation.XmlElementDecl;
import javax.xml.bind.annotation.XmlRegistry;
import javax.xml.namespace.QName;

@XmlRegistry
public class ObjectFactory {

    private final static QName _Player_QNAME = new QName(&quot;http://entity.jaxbtest.ciruk.pl/&quot;, &quot;Player&quot;);

    /**
     * Create a new ObjectFactory that can be used to create new instances of
     * schema derived classes for package: pl.atena.partweb.policytransfer
     *
     */
    public ObjectFactory() {
    }

    public Team createTeam() {
        return new Team();
    }

    /**
     * Create an instance of {@link JAXBElement }{@code &lt;}
     * {@link Player }{@code &gt;}
     *
     */
    @XmlElementDecl(namespace = &quot;&quot;, name = &quot;Player&quot;)
    public JAXBElement&lt;Player&gt; createPlayer(Player value) {
        return new JAXBElement&lt;Player&gt;(_Player_QNAME, Player.class, null, value);
    }
}
[/sourcecode]