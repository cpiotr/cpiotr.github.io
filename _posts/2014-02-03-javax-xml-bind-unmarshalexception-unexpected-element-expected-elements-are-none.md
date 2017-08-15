---
ID: 183
post_title: 'javax.xml.bind.UnmarshalException: unexpected element. Expected elements are (none)'
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2014-02-03 17:11:47
---
While providing a stub for SOAP web service endpoint a need occured to return real-life data from production environment to track a <a href="http://en.wikipedia.org/wiki/Heisenbug" target="_blank">heisenbug</a>.
Such data was available in form of rather spacious XML element. It wasn't a complete document, but a non-root part of SOAP payload.
JAXB Model was generated from XML Schema, however when trying to naively deserialize data aformentioned error popped up: `javax.xml.bind.UnmarshalException: unexpected element (uri:"", local:"Yadayada"). Expected elements are (none)`.

```
String xml =
    "<?xml version=\"1.0\" encoding=\"UTF-8\" standalone=\"yes\"?>" +
    "<Player>" +
        "<firstName>Stiopan</firstName>" +
        "<lastName>Lafayete</lastName>" +
    "</Player>";

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
```

The key point to remember is that complex types from XML schema have their metadata mapped to annotated methods in `ObjectFactory`. Instantiating `JaxbContext `on non-root class does not provide sufficient information for deserialization.
Instead, package name or `ObjectFactory `object must be used to fulfill the requirement.

```
public static <T> String marshall(T entity) throws JAXBException {
    Class<T> entityClass = (Class<T>) entity.getClass();
   
    // At this point ObjectFactory is required within provided package
    JAXBContext jaxbContext = JAXBContext.newInstance(entityClass.getPackage().getName());
   
    Marshaller m = jaxbContext.createMarshaller();
   
    JAXBElement<T> element = new JAXBElement<T>(new QName(entityClass.getSimpleName()), entityClass, entity);
   
    StringWriter sw = new StringWriter();
    m.marshal(element, sw);
   
    return sw.toString();
}

public static <T> T unmarshall(InputStream is, Class<T> entityClass) throws JAXBException {
    // At this point ObjectFactory is required within provided package
    JAXBContext jaxbContext = JAXBContext.newInstance(entityClass.getPackage().getName());
   
    Unmarshaller jaxb = jaxbContext.createUnmarshaller();
    JAXBElement<T> entityElement = (JAXBElement<T>) jaxb.unmarshal(is);
   
    T entity = entityElement.getValue();
    return entity;
}

public static void main(String[] args) {
    String xml =
        "<?xml version=\"1.0\" encoding=\"UTF-8\" standalone=\"yes\"?>" +
        "<Player>" +
            "<firstName>Stiopan</firstName>" +
            "<lastName>Lafayete</lastName>" +
        "</Player>";
   
    Player p = new Player();
    p.setFirstName("Joe");
    p.setLastName("McQeerk");
    try {
        InputStream inputStream = new ByteArrayInputStream(xml.getBytes());
       
        System.out.println(marshall(p));
       
        Player player = unmarshall(inputStream, Player.class);
        inputStream.close();
       
        System.out.println(String.format("\nPlayer name: %s, %s", player.getLastName(), player.getFirstName()));
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

For code completeness, the model information is provided below.

```
package pl.ciruk.jaxbtest.entity;

import javax.xml.bind.annotation.XmlAccessType;
import javax.xml.bind.annotation.XmlAccessorType;
import javax.xml.bind.annotation.XmlElement;
import javax.xml.bind.annotation.XmlType;

@XmlAccessorType(XmlAccessType.FIELD)
@XmlType(name = "Player", namespace = "http://entity.jaxbtest.ciruk.pl/", propOrder = {
    "firstName", "lastName"
})
public class Player {
    @XmlElement(name = "firstName", nillable = true)
    private String firstName;

    @XmlElement(name = "lastName", nillable = true)
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
```

```
package pl.ciruk.jaxbtest.entity;

import java.util.ArrayList;
import java.util.List;

import javax.xml.bind.annotation.XmlAccessType;
import javax.xml.bind.annotation.XmlAccessorType;
import javax.xml.bind.annotation.XmlElement;
import javax.xml.bind.annotation.XmlRootElement;
import javax.xml.bind.annotation.XmlType;

@XmlAccessorType(XmlAccessType.FIELD)
@XmlRootElement(name = "Team")
@XmlType(name = "Team", namespace = "http://entity.jaxbtest.ciruk.pl/", propOrder = {
    "players"
})
public class Team {
   
    @XmlElement(name = "players", nillable = true)
    private List<Player> players;

    public void setPlayers(List<Player> players) {
        this.players = players;
    }

    public List<Player> getPlayers() {
        if (players == null) {
            players = new ArrayList<Player>();
        }
        return players;
    }
}
```

```
package pl.ciruk.jaxbtest.entity;

import javax.xml.bind.JAXBElement;
import javax.xml.bind.annotation.XmlElementDecl;
import javax.xml.bind.annotation.XmlRegistry;
import javax.xml.namespace.QName;

@XmlRegistry
public class ObjectFactory {

    private final static QName _Player_QNAME = new QName("http://entity.jaxbtest.ciruk.pl/", "Player");

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
     * Create an instance of {@link JAXBElement }{@code <}
     * {@link Player }{@code >}
     *
     */
    @XmlElementDecl(namespace = "", name = "Player")
    public JAXBElement<Player> createPlayer(Player value) {
        return new JAXBElement<Player>(_Player_QNAME, Player.class, null, value);
    }
}
```