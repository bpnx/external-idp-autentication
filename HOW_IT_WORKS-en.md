# FOREWORD #

The following solution is published "as is" with no warranty whatsoever. It is intended for people with good knowledge of the subject who intend to adapt it to their reality.

# The problem #

We have faced the issue of how to offer the user the choice to authenticate against her institutional IdP as well as against one ore more external IdPs.

* allow users to choose an IdP, other from their institutional, to authenticate against
* integrate the attributes received from the external IdP with the ones normally released by the institutional Idp
* leave the SPs unmodified, letting them communicate as usual with the institutional Idp
* avoid as much as possible programming for the sake of maintenance semplicity. Use of existing features preferred when possible

Software used:

* Shibboleth IDP 3.3.1
* Shibboleth SP 2.6
* Apache 2.4 (it's probably possible to adapt the technique to other systems)

# How it works #

This solution implies reconfiguring the IdP and adding some software to it.

Some naming conventions. **_IDPint_** is the reconfigured institutional IdP, which allows authenticating both locally and towards an external IdP ; **_IDPext_** is the external IdP. In the given solution the shibboleth IdP software and the shibboleth SP software are installed on the same server hosting **IDPint**: the SP service will be called **_SPidpint_**. **SPidpint** and **IDPint** are installed the same physical server.

**Password** authentication is used for local authentication on **IDPint**, whereas the authentication mechanism used by **IDPext** is not known in advance.

## The flow ##

The following is a simplified authentication flow which can be obviously extended

1. The user wants to connect to the service hosted on **SP1**
1. SP1 redirects the user to **IDPint**
1. The normal login page is added with a button to allow authentication against an external IdP
1. If the user types in her username and password she authenticates as usual, locally on **IDPint**
1. Otherwise, by cliccking on the button, she will be redirected to authenticate on **IDPext**
1. Once authenticated against **IDPext** the user returns to **SPint** and gets then redirected to **SP1**, carrying both the **IdPext** **and** (if user exists on IDPint) the **IDPint attributes**, merged together as decided by the **IDPint** system administrator.

Attributes can obviously be merged only if a unique association between **IDPext** and **IDPint** identities exist.

### Considerazioni ###

Foreword: according to SAML specs it is necessary that the user's browser gets redirected to **IDPext** to allow the user to authenticate towards **IDPext**; this operation is normally performed by the Service Provider

The problem to solve is therefore:

* have the user authenticate towards **IDPext**
* get the user return to the **SP1** initially accessed
* have the user pass through **IDPint**, after successful authentication and the attribute release by **IDPext**, to receive also the attributes from **IDPint**
* not have the user authenticate against **IDPint** AS WELL, after successful authentication against **IDPext**

## Implementation ##

First of all:

* install Apache on **IDPint**, configured a as reverse proxy (i.e. authentication requests get to Apache. Apache turns them to the IDP software running on localhost)
* install shibboleth SP software on **IDPint** (with the apache module)

Authentication against **IDPext** will occur via the authn/RemoteUser flow, called by the _authn/MFA_ flow.

### Apache configuration ###

The Apache instance has a double role: reverse proxy in front of **IDPint** and SP to start the **IDPext** authentication. 
One of the key points of this solution is that the SP software is triggered when user selects the external authentication: this is achieved by protecting the authentication path used by the flow authn/RemoteUser with the Shibboleth SP software. By clicking on the external authentication button the user causes the flow to goes under the control of **SPidpint**

The default authn/RemoteUser flow Location is:

  /idp/Authn/RemoteUser

The (partial) Apache configuration for the "/idp/Authn/RemoteUser" Location is:

```apache
 <Location "/idp/Authn/RemoteUser">
        AuthType shibboleth
        ShibRequestSetting requireSession true
        Require shibboleth
 </Location>
```

Upon successful authentication the control goes back to **IDPint**, that needs to get a "Remote User" value. To let Apache pass the server REMOTE_USER variable to **IDPint** we have chosen to use the "Headers" Apache module ("RequestHeader" directive) since the authn/RemoteUser flow supports reading the "Remote User" value from an HTTP header.

To sum up:

1. when the user selects external authentication on **IDPext** 
1. the Shibboleth SP software is triggered, 
1. and sends the user to the external IDP.
1. After successful authentication the user gets back to the **IDPint**, ending the authn/RemoteUser flow, 
1. and **IDPint** will get the "Remote User" from an HTTP header, created by the Apache instance (RequestHeader directive)

So the complete Apache configuration for the "/idp/Authn/RemoteUser" Location is: 

```apache
  <Location "/idp/Authn/RemoteUser">
        AuthType shibboleth
        ShibRequestSetting requireSession true
        Require shibboleth
        
        # send REMOTE_USER to IDPint:
        RequestHeader unset Idpext-Id
        RequestHeader set Idpext-Id expr=%{REMOTE_USER}
  </Location>
```

Still, there's a problem. Attributes released by **IDPext** are lost, because Shibboleth SP and IDP instances, installed on the same server, are not able to exchange information natively. But thanks to Apache there's a simple solution: using the "RequestHeader" directive, as we did for the REMOTE_USER server variable, it's possible to send attributes released by **IDPext** to the **IDPint** instance (remember that the SP instance converts all attributes received into Apache variables following the mapping given in attribute-map.xml). During the attribute resolution phase the **IDPint** can then access the HTTP headers using the object:

  shibboleth.HttpServletRequest

So the Apache configuration is (note "requireSession false"):

```apache
  <Location "/idp/profile">
        AuthType shibboleth
        ShibRequestSetting requireSession false
        Require shibboleth
        
    # Send the attributes released by IDPext to IDPint making use of HTTP headers
    # idpext_attribute1, idpext_attribute2, etc. are the names of the attributes, as mapped in attribute-map.xml on the SP 
    # 
    
	# RequestHeader unset Idpext-Id 
	# RequestHeader set Idpext-Id expr=%{REMOTE_USER}

	RequestHeader unset Idpext-Attribute1
	RequestHeader set Idpext-Attribute1 expr=%{reqenv:idpext_attribute1}

	RequestHeader unset Idpext-Attribute2
	RequestHeader set Idpext-Attribute2 expr=%{reqenv:idpext_attribute2}

	RequestHeader unset Idpext-Attribute3
	RequestHeader set Idpext-Attribute3 expr=%{reqenv:idpext_attribute3}

	RequestHeader unset Idpext-Attribute4
	RequestHeader set Idpext-Attribute4 expr=%{reqenv:idpext_attribute3}
  </Location>
```

In the given example idpext_attribute1 is the name of the env varable generated by the mod_shib2 Apache module
(identical to the name set in attribute-map.xml), whereas Idpext-Attribute1 is the name of the header generated by Apache.

**Important note** : this solution makes use of HTTP headers to transfer attributes and REMOTE_USER, which will be used to obtain the final Principal Name. Attention must be paid to avoid that someone can use "http header forgery" to impersonate someone else: it is therefore OF THE UTMOST IMPORTANCE to always unset all used headers. Moreover it's good politics to choose non trivial header names to make possible forgery attacks harder.

### IDPint configuration ###

First of all it is necessary to configure the IDP software on **IDPint** to have it read the header containing the REMOTE_USER (see the authn/RemoteUser documentation); in this example the HTTP header considered is "Idpext-Id". The file to modify is **web.xml**:

```xml
    <!-- Servlet protected by container used for RemoteUser authentication -->
    <servlet>
        <servlet-name>RemoteUserAuthHandler</servlet-name>
        <servlet-class>net.shibboleth.idp.authn.impl.RemoteUserAuthServlet</servlet-class>
        <init-param>
           <!-- to enable header checking -->
            <param-name>checkHeaders</param-name>
            <param-value>Idpext-Id</param-value>
           <!-- to enable header checking --> 
        </init-param>
        <load-on-startup>2</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>RemoteUserAuthHandler</servlet-name>
        <url-pattern>/Authn/RemoteUser</url-pattern>
    </servlet-mapping>
```    

The authn/MFA (Multi Factor Authentication) flow is used for authentication since it makes it possible to specify an and-or logic to make use of more authentication methods.
The MFA configuration is the following (slightly adapte from [Shibboleth Wiki](https://wiki.shibboleth.net/confluence/display/IDP30/MultiFactorAuthnConfiguration)):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:util="http://www.springframework.org/schema/util"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:c="http://www.springframework.org/schema/c"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
                           http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
                           http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util.xsd"
                           
       default-init-method="initialize"
       default-destroy-method="destroy">
    
<util:map id="shibboleth.authn.MFA.TransitionMap">

	<entry key="">
        <bean parent="shibboleth.authn.MFA.Transition" p:nextFlow="authn/Password" />
    </entry>

    <entry key="authn/Password">
        <bean parent="shibboleth.authn.MFA.Transition"> 
            <property name="nextFlowStrategyMap">
                <map>
                    <entry key="ChooseMethodB" value="authn/RemoteUser" />
                </map>
            </property>
        </bean>
    </entry> 
 
    <entry key="authn/RemoteUser"  >
        <bean parent="shibboleth.authn.MFA.Transition"/>
    </entry>

</util:map>

</beans>
```

The **ChooseMethodB** event is declared in the following file:

**{idp.home}/conf/authn/authn-events-flow.xml**:
```xml

    <end-state id="ChooseMethodB" />

    <global-transitions>
        <transition on="ChooseMethodB" to="ChooseMethodB" />
    </global-transitions>

```

With this configuration:

1. the "authn/Password" flow is called (which in turn calles the **login.vm** view)
1. if the **proceed** events is returned (generated by clicking on the original authentication button of the login.vm view) the the flow ends and the attribute resolution phase is entered
1. otherwise, if the **ChooseMethodB** event is returned then the "authn/RemoteUser" flow is called, regarding external authentication

The **login.vm** view has been modified so to return, besides the classical "proceed" event, the **ChooseMethodB** event (in case the user click on the external authentication button)

Define headers on  **IDPint** to merge attributes given from **IDPext** with **IDPint** attributes: su **IDPint**:

```xml
        <AttributeDefinition id="idpext-attribute1" xsi:type="ScriptedAttribute" customObjectRef="shibboleth.HttpServletRequest">
                <AttributeEncoder xsi:type="SAML2String" name="idpext-attribute1" friendlyName="idpext-attribute1" />
                <Script><![CDATA[       
                if (custom.getHeader("Idpext-Attribute1") != ""  && custom.getHeader("Idpext-Attribute1") != "(null)" ) {
                        idpext-attribute1.addValue(custom.getHeader("Idpext-Attribute1"));
                }
        ]]></Script>
        </AttributeDefinition>
```

### SPidpint configuration ###

* the **SPidpint** service must be configured to use **IDPext** as IDP. Obviously it's also possible to use a discovery service to support more external IDPs

* configure attribute-map.xml in the preferred way

Note: the REMOTE_USER value after a successful **IDPext** authentication, depends on the **SPidpint** shibboleth2.xml configuration

## Multiple identities ##

It is possible that an authenticated **IDPext** user has more than one identity on **IDPint**; we used a workaround for now, administratively forcing the user identity on **IDPint**, using our rules. This is not a big problem for us, because users can always authenticate directly on **IDPint** choosing the preferred identity. We have followed examples from [Shibboleth Wiki](https://wiki.shibboleth.net/confluence/display/IDP30/AttributePostLoginC14NConfiguration#AttributePostLoginC14NConfiguration-OneCommonUseofAttribute-basedCanonicalization) , mainly the configuration examples of the file **attribute-sourced-subject-c14n-config.xml**

## SPID integration ##

SPID ("Sistema Pubblico di Identità Digitale") is the italian public authentication system.

Using the method described above the integration with SPIS is quite straitforward.
**IDPext** represents one of the SPID IdPs; to adhere to the governmental guidelines on the web interface we have slightly modified the Shibboleth Embedded Discovery Service, in order to use cookies for the IdP choice.
By choosing an IdP in the login page a cookie gets set with the entity ID of the chosen IdP.
The modified EDS uses the set cookie to connect to the IdP.

# Autori #

Marco Naimoli (<marco.naimoli@unipd.it>), Stefano Zanmarchi (<stefano.zanmarchi@unipd.it>)