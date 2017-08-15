---
ID: 191
post_title: Maven behind a proxy
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2014-04-08 20:40:55
---
Corporate policies often require tunneling all Internet traffic through a proxy. Such situation makes the default maven setup useless in terms of access to remote repositories. 
Custom settings must be introduced in order to get maven working again.
Configuration descriptor should be present at `{M2_HOME}/conf/settings.xml`, which typically resolves to `~/.m2/conf/settings.xml`. Sample basic file is presented below.
It is important to note, that in case of <a href="http://en.wikipedia.org/wiki/NT_LAN_Manager" target="_blank">NTLM</a> proxy, the domain name to which the user belongs may have to be explictly set, i.e.
```
<username>DOMAIN\username</username>
```

```
<settings>
	<proxies>
		<proxy>
			<active>true</active>
			<protocol>http</protocol>
			<host>proxy.domain.com</host>
			<port>8080</port>
			<username>DOMAIN\username</username>
			<password>pass</password>
			<nonProxyHosts>*.google.com|*.mycompany.com</nonProxyHosts>
		</proxy>
	</proxies>
</settings>
```

It is possible that after those changes Eclipse Maven plugin still does not fetch data from remote repositories, and hence suggest nothing. Maven repository index needs to be enabled and rebuild.
To achieve that open the <em>Maven Repositories</em> view, select <em>Global Repositories</em>, right-click on chosen repository and enable either full or minimum index and rebuild it.