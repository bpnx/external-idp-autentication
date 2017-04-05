# PREMESSA #

Quella delineata è la nostra soluzione; è resa pubblica a puro scopo di condivisione, ma si declina ogni reponsabilità per effetti imprevisti. E' destinato ad un pubblico che abbia già dimestichezza con l'argomento e sia in grado di adattare l'idea alla propria realtà.

# Il problema #

Il problema che abbiamo dovuto affrontare è stato quello di permettere all'utente di scegliere se autenticarsi presso l'IDP istituzionale o uno o più IDP esterni.

* utilizzare, a scelta dell'utente, un secondo IDP, diverso da quello istituzionale, per l'autenticazione
* integrare gli attributi dell'IDP esterno con quello istituzionale
* non modificare la configurazione degli attuali SP, che si rivolgono unicamente all'IDP istituzionale
* non utilizzare, il piu' possibile, programmazione in alcun linguaggio (un programma, anche scritto "in casa", ha un suo costo in termini di impegno per la manutenzione) e utilizzare di preferenza l'utilizzo di qualcosa di già esistente

Quanto abbiamo utilizzato utilizza:

* Shibboleth IDP 3.3.1
* Shibboleth SP 2.6
* Apache 2.4 (ma è probabilmente possibile adattare la tecnica ad altri sistemi)

# Come funziona #

La realizzazione pratica implica la riconfigurazione dell'IDP, con l'aggiunta di software

In questa spiegazione **_IDPint_** rappresenta l'IDP istituzionale riconfigurato per permettere sia l'autenticazione locale, sia l'utilizzo di un IDP esterno; **_IDPext_** rappresenta l'IDP esterno. Nella soluzione, come si vedrà, nello stesso server che ospita **IDPint** verrà installato Shibboleth SP: il servizio di SP, in questo caso, verrà chiamato **_SPidpint_**. Quindi **SPidpint** e **IDPint** sono lo stesso server fisico.

L'autenticazione utilizzata per l'autenticazione locale su **IDPint** è **Password**, mentre quella utilizzata per l'autenticazione presso **IDPext** non è nota a priori.

## Il flusso ##

Di seguito il flusso di autenticazione, semplificato; puo' ovviamente essere esteso

1. L'utente vuole connettersi al servizio **SP1**
1. SP1 lo rimanda a **IDPint**
1. L'utente si trova di fronte alla classica schermata di autenticazione con Password, ma ha a disposizione un pulsante aggiuntivo per l'autenticazione esterna
1. Se utilizza username e password, l'autenticazione avviene come di consueto, in locale su **IDPint**
1. Se sceglie il pulsante di autenticazione esterna , viene spedito ad **IDPext**, presso il quale può autenticarsi
1. Una volta autenticato presso **IDPext**, torna al servizio **SP1**, con gli **attributi di IDPext** + gli **attributi di IDPint** (se l'utente esiste anche su **IDPint**), fusi in base ad una logica stabilita dall'amministratore di sistema di **IDPint**

Ovviamente per realizzare la fusione degli attributi, deve già esistere un'associazione univoca da identità dell'utente su **IDPext** ed identità su **IDPint**.

### Considerazioni ###

[DA RIVEDERE, forse non serve ] Premessa: per fare in modo che l'utente si autentichi presso **IDPext**, secondo il paradigma SAML, è necessario che il browser dell'utente venga mandato all'**IDPext** tramite redirect; questa normalmente è l'operazione svolta dal software di Service Provider

Il problema da risolvere è quindi:

* far autenticare l'utente su **IDPext**
* far tornare l'utente al proprio **SP1** originario
* far in modo che l'utente "passi" anche attraverso **IDPint**, dopo l'autenticazione e il rilascio di attributi da parte di **IDPext**, per ricevere anche gli attributi di **IDPint**
* non richiedere all'utente di autenticarsi ANCHE su **IDPint**, dopo essersi già autenticato su **IDPext**

## Implementazione ##

Innanzitutto:

* installare Apache su **IDPint**, configurato come reverse proxy di **IDPint** (quindi le richieste di autenticazione arrivano su Apache, che le rigira al software di IDP in localhost)
* installare il software Shibboleth SP su **IDPint** (con modulo per Apache)

L'autenticazione tramite **IDPext** avverrà tramite il flow _authn/RemoteUser_; verrà richiamato tramite il flow _authn/MFA_.

### Configurazione Apache ###

Apache ha il doppio ruolo di reverse proxy di **IDPint** ed SP per l'autenticazione su **IDPext**. 
Uno dei punti cruciali della soluzione è che la parte SP "scatti" quando viene selezionata l'autenticazione esterna: quindi è necessario proteggere la Location di autenticazione RemoteUser con il software di Shibboleth SP. In questo modo, quando l'utente preme il pulsante per l'autenticazione esterna, passa inconsapevolmente sotto il controllo di **SPidpint**

La Location di default relativa al flusso authn/RemoteUser è:

  /idp/Authn/RemoteUser

```apache
 <Location "/idp/Authn/RemoteUser">
        AuthType shibboleth
        ShibRequestSetting requireSession true
        Require shibboleth
 </Location>
```

Per trasferire i dati relativi al REMOTE_USER e gli attributi ricevuti all'**IDPint** è necessario utilizzare il modulo Apache "headers" e configurare degli appositi header (RequestHeader).

Riassumendo:

1. quando l'utente seleziona l'autenticazione su **IDPext** 
1. viene attivato lo Shibboleth SP, 
1. che porta l'utente ad autenticarsi presso un IDP esterno. 
1. Dopo l'autenticazione, l'utente "ritorna" al path iniziale, "/idp/Authn/RemoteUser", proseguendo l'autenticazione authn/RemoteUser. 
1. L'idp (**IDPint**) utilizzerà come REMOTE_USER il valore di un particolare header http, creato da Apache tramite la direttiva RequestHeader

La configurazione completa di Apache per la Location "/idp/Authn/RemoteUser": 

```apache
  <Location "/idp/Authn/RemoteUser">
        AuthType shibboleth
        ShibRequestSetting requireSession true
        Require shibboleth
        
        # La seguente direttiva manda il REMOTE_USER all'IDPint
        RequestHeader unset Idpext-Id
        RequestHeader set Idpext-Id expr=%{REMOTE_USER}
  </Location>
```

C'è però un problema: gli attributi rilasciati da **IDPext** vengono persi, perchè Shibboleth SP e Shibboleth IDP sono installate nello stesso computer ma non dialogano fra di loro direttamente; grazie all'istanza di Apache installata c'è una semplice soluzione: analogamente a quanto fatto con REMOTE_USER, è possibile mandare gli attributi ricevuti da **IDPext**, tramite la direttiva RequestHeader, ad **IDPint**; su **IDPint**, grazie all'oggetto

  shibboleth.HttpServletRequest

è possibile leggere gli header HTTP interessati (o probabilmente costruire un Connector per estrarli).

La configurazione di Apache è quindi (notare il "requireSession false"):

```apache
  <Location "/idp/profile">
        AuthType shibboleth
        ShibRequestSetting requireSession false
        Require shibboleth
        
    # Le righe successive trasportano il valore degli attributi ricevuti da IDPext
    # rendendoli disponibili, come header, a IDPint
    # idpext_attribute1, idpext_attribute2, ecc.ecc. sono i nomi degli attributi, mappati secondo attribute-map.xml dell'SP 
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

In questo esempio idpext_attribute1 è il nome della variabile di environment generata dal modulo di apache
mod_shib2 (che coincide con quanto configurato su attribute-map.xml), mentre Idpext-Attribute1 è il nome del
RequestHeader generato da Apache

**Nota importante** : questa soluzione utilizza gli header HTTP per trasportare i dati degli attributi e del REMOTE_USER, che contribuiranno a ricavare il Principal Name finale; vanno quindi utilizzate delle accortezze per evitare che un utente possa "forgiare" header per impersonare terzi: a questo scopo è FONDAMENTALE azzerare sempre tutti gli header coinvolti; inoltre è bene utilizzare nomi di header non ovvi, in modo da rendere più complicata l'impresa ad un eventuale intruso.

### Configurazione IDPint ###

Innanzitutto è necessario configurare anche la parte IDP di **IDPint** per fare in modo che il software di IDP possa recuperare l'header dal quale ricavare il REMOTE_USER (cfr. documentazione authn/RemoteUser); in questo esempio l'header HTTP preso in considerazione è "Idpext-Id". Il file da modificare è **web.xml**:

```xml
    <!-- Servlet protected by container used for RemoteUser authentication -->
    <servlet>
        <servlet-name>RemoteUserAuthHandler</servlet-name>
        <servlet-class>net.shibboleth.idp.authn.impl.RemoteUserAuthServlet</servlet-class>
        <init-param>
           <!-- permette di riconoscere gli headers -->
            <param-name>checkHeaders</param-name>
            <param-value>Idpext-Id</param-value>
           <!-- permette di riconoscere gli headers --> 
        </init-param>
        <load-on-startup>2</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>RemoteUserAuthHandler</servlet-name>
        <url-pattern>/Authn/RemoteUser</url-pattern>
    </servlet-mapping>
```    

Per l'autenticazione viene utilizzato il flow authn/MFA (Multi Factor Authentication), grazie al quale è possibile specificare una logica per l'utilizzo di più metodi di autenticazione.
La configurazione MFA è la seguente (copiata da Shibboleth Wiki e leggermente modificata):

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

L'evento **ChooseMethodB** viene inoltre dichiarato nel file:

**{idp.home}/conf/authn/authn-events-flow.xml**:
```xml

    <end-state id="ChooseMethodB" />

    <global-transitions>
        <transition on="ChooseMethodB" to="ChooseMethodB" />
    </global-transitions>

```

In questa configurazione:

1. viene chiamato il flow "authn/Password" (che a sua volta richiama la view **login.vm**
1. se viene ritornato l'evento **proceed** (generato dalla pressione dell'originario pulsante di autenticazione della view login.vm) allora il flusso si conclude e si passa alla fase della risoluzione attributi
1. se viene ritornato invece l'evento **ChooseMethodB** viene chiamato il flow "authn/RemoteUser", che riguarda l'autenticazione esterna

La view **login.vm** è stata modificata in modo da ritornare, oltre al classico evento "proceed", l'evento **ChooseMethodB** (nel caso in cui l'utente selezioni il pulsante di autenticazione su IDP esterno)

Per estrarre gli header con i dati degli attributi di **IDPext** ed integrarli con quelli di **IDPint**, su **IDPint** posso definire attributi di questo tipo:

```xml
        <resolver:AttributeDefinition id="idpext-attribute1" xmlns="urn:mace:shibboleth:2.0:resolver:ad" xsi:type="ad:Script" customObjectRef="shibboleth.HttpServletRequest">
                <resolver:AttributeEncoder xsi:type="enc:SAML2String" name="idpext-attribute1" friendlyName="idpext-attribute1" />
                <Script><![CDATA[       
                if (custom.getHeader("Idpext-Attribute1") != ""  && custom.getHeader("Idpext-Attribute1") != "(null)" ) {
                        idpext-attribute1.addValue(custom.getHeader("Idpext-Attribute1"));
                }
                
        ]]></Script>
        </resolver:AttributeDefinition>
```

### Configurazione SPidpint ###

* il servizio **SPidpint** deve essere configurato appositamente in modo da utilizzare **IDPext** come IDP di autenticazione. Si può ovviamente utilizzare un discovery service, in modo da poter utilizzare in modo flessibile più di un IDP esterno

* configurare attribute-map.xml in modo opportuno

Nota: l'autenticazione tramite **IDPext**, valorizza, di fatto, la variabile REMOTE_USER in base alla configurazione shibboleth2.xml di **SPidpint**


## Integrazione SPID ##

SPID è il "Sistema Pubblico di Identità Digitale" lanciato dal governo italiano.

Utilizzando il metodo appena descritto, l'integrazione con SPID a questo punto è abbastanza evidente, **IDPext** rappresenta uno degli Identity Provider SPID; 
nella realizzazione abbiamo modificato leggermente lo Shibboleth Embedded Discovery Service, in modo da poter utilizzare i cookie come metodo di scelta dell'IDP: in questo modo 
nella maschera di login, la selezione dell'IDP da parte dell'utente causa la scrittura di un cookie con l'entity ID dell'IDP prescelto; l'EDS modificato utilizza tale cookie
per la connessione all'IDP.

# Autori #

Marco Naimoli (<marco.naimoli@unipd.it>), Stefano Zanmarchi (<stefano.zanmarchi@unipd.it>)
