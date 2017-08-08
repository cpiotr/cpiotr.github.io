---
ID: 191
post_title: Maven behind a proxy
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2014/04/maven-behind-a-proxy/
published: true
post_date: 2014-04-08 20:40:55
---
Corporate policies often require tunneling all Internet traffic through a proxy. Such situation makes the default maven setup useless in terms of access to remote repositories. 
Custom settings must be introduced in order to get maven working again.
Configuration descriptor should be present at <code>{M2_HOME}/conf/settings.xml</code>, which typically resolves to <code>~/.m2/conf/settings.xml</code>. Sample basic file is presented below.
It is important to note, that in case of <a href="http://en.wikipedia.org/wiki/NT_LAN_Manager" target="_blank">NTLM</a> proxy, the domain name to which the user belongs may have to be explictly set, i.e.
[sourcecode lang="XML"]
&lt;username&gt;DOMAIN\username&lt;/username&gt;
[/sourcecode]

[sourcecode lang="XML"]
&lt;settings&gt;
	&lt;proxies&gt;
		&lt;proxy&gt;
			&lt;active&gt;true&lt;/active&gt;
			&lt;protocol&gt;http&lt;/protocol&gt;
			&lt;host&gt;proxy.domain.com&lt;/host&gt;
			&lt;port&gt;8080&lt;/port&gt;
			&lt;username&gt;DOMAIN\username&lt;/username&gt;
			&lt;password&gt;pass&lt;/password&gt;
			&lt;nonProxyHosts&gt;*.google.com|*.mycompany.com&lt;/nonProxyHosts&gt;
		&lt;/proxy&gt;
	&lt;/proxies&gt;
&lt;/settings&gt;
[/sourcecode]

It is possible that after those changes Eclipse Maven plugin still does not fetch data from remote repositories, and hence suggest nothing. Maven repository index needs to be enabled and rebuild.
To achieve that open the <em>Maven Repositories</em> view, select <em>Global Repositories</em>, right-click on chosen repository and enable either full or minimum index and rebuild it.