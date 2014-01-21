---
layout: post
title: "Jetty源码阅读 应用部署"
description: ""
category: ""
tags: [Jetty]
---
{% include JB/setup %}
Jetty 应用部署管理器是由在Jetty启动时加入Server的
具体配置在jetty-deploy.xml中

{% highlight xml linenos %}
<Call name="addBean">
  <Arg>
    <New id="DeploymentManager" class="org.eclipse.jetty.deploy.DeploymentManager">
      <Set name="contexts">
        <Ref refid="Contexts" />
      </Set>
      <Call name="setContextAttribute">
        <Arg>org.eclipse.jetty.server.webapp.ContainerIncludeJarPattern</Arg>
        <Arg>.*/servlet-api-[^/]*\.jar$</Arg>
      </Call>

      <Call id="webappprovider" name="addAppProvider">
        <Arg>
          <New class="org.eclipse.jetty.deploy.providers.WebAppProvider">
            <Set name="monitoredDirName"><Property name="jetty.base" default="." />/webapps</Set>
            <Set name="defaultsDescriptor"><Property name="jetty.home" default="." />/etc/webdefault.xml</Set>
            <Set name="scanInterval">1</Set>
            <Set name="extractWars">true</Set>
            <Set name="configurationManager">
              <New class="org.eclipse.jetty.deploy.PropertiesConfigurationManager">
                <!-- file of context configuration properties
                <Set name="file"><SystemProperty name="jetty.base"/>/etc/some.properties</Set>
                -->
                <!-- set a context configuration property
                <Call name="put"><Arg>name</Arg><Arg>value</Arg></Call>
                -->
              </New>
            </Set>
          </New>
        </Arg>
      </Call>
    </New>
  </Arg>
</Call>
{% endhighlight %}

![Jetty Deploy]({{ BASE_PATH }}/assets/images/JettyDeploy.png)

web.xml会在WebXmlConfiguration中初始化.结果会到WebAppContext的_metaData中.并使用WebDescriptor来初始化。
使用annotation时，需要调用AnnotationConfiguration的scanForAnnotations方法扫描annotation.
这个方法会调用asm库来扫描annotation.扫描结果会加入到WebAppContext的_metaData中。这个对象将在WebAppContext的startContext中初始化.


web.xml会在WebXmlConfiguration中初始化.结果会加入到WebAppContext的_metaData中
