knoxplorer
=======
Browse HDFS through Apache Knox.

A simple demo application that leverages KnoxSSO service and SSOCookie provider to authenticate once and use the SSO cookie for subsequent requests. This article also illustrates the new application hosting feature in Knox 0.9.0.

![alt text](knoxplorer.png "KnoXplorer")

1. Clone or checkout this project inside the {GATEWAY_HOME}/data/applications directory of your Knox installation.
2. Copy sandbox-apps.xml from {GATEWAY_HOME}/templates and add the application definition for knoxplorer2
```
<application>
    <name>knoxplorer2</name>
</application>
```

Navigate to https://c6401.ambari.apache.org:8443/gateway/sandbox-apps/knoxplorer2/index.html

Login Details and Link
========
The SSOCookieProvider is used to protect access to the knoxplorer application and each of the services that would be consumed by the applications hosted within this topology definition.

SSOCookie provider is configured to redirect any requests that do not present the expected cookie to the following URL:
https://c6401.ambari.apache.org:8443/gateway/knoxsso/api/v1/websso

It adds the actually requested URL as the origninalUrl query parameter in the redirect. This indicates to the KnoxSSO service the URL to which the browser needs to be redirected again following authentication.

It also provides a query parameter for the knoxplorer application that indicates the topology to use as the endpoint for Hadoop cluster access and is currently hardcoded to "sandbox".

*NOTE:* ALL of the URLs in this application and topology examples reference c6401.ambari.apache.orf as this article assumes testing with an Ambari quickstart cluster with Apache Knox 0.9.0 installed via the shared /vagrant directory.

Apache Knox Configuration
========

## KnoxSSO Topology:

In order for the Login Link above to work, we need to have a configured KnoxSSO topology in the Knox Gateway instance that we are point to. Below is an example that leverages an Okta application for SSO:

```
<topology>
    <gateway>
      <provider>
        <role>webappsec</role>
        <name>WebAppSec</name>
        <enabled>true</enabled>
        <param>
            <name>xframe.options.enabled</name>
            <value>true</value>
        </param>
      </provider>
        <provider>
            <role>authentication</role>
            <name>ShiroProvider</name>
            <enabled>true</enabled>
            <param>
                <name>sessionTimeout</name>
                <value>30</value>
            </param>
            <param>
                <name>redirectToUrl</name>
                <value>/gateway/knoxsso/knoxauth/login.html</value>
            </param>
            <param>
                <name>restrictedCookies</name>
                <value>rememberme,WWW-Authenticate</value>
            </param>
            <param>
                <name>main.ldapRealm</name>
                <value>org.apache.hadoop.gateway.shirorealm.KnoxLdapRealm</value>
            </param>
            <param>
                <name>main.ldapContextFactory</name>
                <value>org.apache.hadoop.gateway.shirorealm.KnoxLdapContextFactory</value>
            </param>
            <param>
                <name>main.ldapRealm.contextFactory</name>
                <value>$ldapContextFactory</value>
            </param>
            <param>
                <name>main.ldapRealm.userDnTemplate</name>
                <value>uid={0},ou=people,dc=hadoop,dc=apache,dc=org</value>
            </param>
            <param>
                <name>main.ldapRealm.contextFactory.url</name>
                <value>ldap://localhost:33389</value>
            </param>    
            <param>
                <name>main.ldapRealm.authenticationCachingEnabled</name>
                <value>false</value>
            </param>
            <param>
                <name>main.ldapRealm.contextFactory.authenticationMechanism</name>
                <value>simple</value>
            </param>
            <param>
                <name>urls./**</name>
                <value>authcBasic</value>
            </param>
        </provider>
                <provider>
            <role>identity-assertion</role>
            <name>Default</name>
            <enabled>true</enabled>
        </provider>
    </gateway>

    <application>
      <name>knoxauth</name>
    </application>

    <service>
        <role>KNOXSSO</role>
        <param>
            <name>knoxsso.cookie.secure.only</name>
            <value>false</value>
        </param>
        <param>
            <name>knoxsso.token.ttl</name>
            <value>30000</value>
        </param>
        <param>
           <name>knoxsso.redirect.whitelist.regex</name>
           <value>^https?:\/\/(www\.local\.com|c6401\.ambari\.apache\.org|localhost|127\.0\.0\.1|0:0:0:0:0:0:0:1|::1):[0-9].*$</value>
        </param>
    </service>
</topology>
```

This particular KnoxSSO deployment is using the Default IDP for form-based SSO authentication against LDAP/AD. This ShiroProvider is used for HTTP Basic Auth credentials against LDAP/AD indentity stores.

You will also notice the KNOXSSO service declaration. Given a successful login by the shiro provider, the KNOXSSO service will create an SSO cookie that can be used as the proof of authentication to the originalUrl until the cookie or the token within the cookie expires. Docs can be found http://knox.apache.org/books/knox-0-9-0/user-guide.html#KnoxSSO+Setup+and+Configuration - for the KnoxSSO service.

What may be less clear is that this simple application doesn't actually care about the user identity but the REST API calls to WebHDFS within the knox.js file of this application must provide the SSO cookie on each call. The cookie is verified by the Apache Knox Gateway for each call to ensure that it is trusted, not expired and extracts the identity information for interaction with the Hadoop WebHDFS service.

Please also note the knoxsso.redirect.whitelist.regex parameter in the KNOXSSO service. This semicolon separated list of regex expressions (there is only one in this case) will be used to validate the originalUrl query parameter to ensure that KnoxSSO will only redirect browsers to trusted sites. This is to avoid things like phishing attacks.

## Sandbox-apps Topology:
The topology that defines the endpoint used to actually access Hadoop resources through the Apache Gateway in this deployment is called sandbox.xml. The following configuration assumes the use of the Hortonworks sandbox VM based Hadoop cluster to enable quick deployment and getting started with Hadoop and app development.

In order to leverage the single sign on capabilities described earlier, this topology much configure the SSOCookie federation provider. This essentially means that the SSO cookie is required in order to access any of the Hadoop endpoints configured within this topology.

```
<topology>
    <gateway>
      <provider>
          <role>federation</role>
          <name>SSOCookieProvider</name>
          <enabled>true</enabled>
          <param>
              <name>sso.authentication.provider.url</name>
              <value>https://c6401.ambari.apache.org:8443/gateway/knoxsso/api/v1/websso</value>
          </param>
      </provider>
      <provider>
          <role>identity-assertion</role>
          <name>Default</name>
          <enabled>true</enabled>
      </provider>
    </gateway>
    <service>
        <role>NAMENODE</role>
        <url>hdfs://localhost:8020</url>
    </service>
    <service>
        <role>JOBTRACKER</role>
        <url>rpc://localhost:8050</url>
    </service>
    <service>
        <role>WEBHDFS</role>
        <url>http://localhost:50070/webhdfs</url>
    </service>
    <service>
        <role>WEBHCAT</role>
        <url>http://localhost:50111/templeton</url>
    </service>
    <service>
        <role>OOZIE</role>
        <url>http://localhost:11000/oozie</url>
    </service>
    <service>
        <role>WEBHBASE</role>
        <url>http://localhost:60080</url>
    </service>
    <service>
        <role>HIVE</role>
        <url>http://localhost:10001/cliservice</url>
    </service>
    <service>
        <role>RESOURCEMANAGER</role>
        <url>http://localhost:8088/ws</url>
    </service>
</topology>
```

Since Knox is hosting knoxplorer in this case, there is no need to enable CORS. Therefore, it is not included in the WebAppSec provider. We do however protect the login page from clickjacking by enabling the xframe-options feature.

Troubleshooting
=======

The most likely trouble that you will run into will be related to cookies and domains. Make sure that the domains that you are using for each configured URL are the same and are acceptable to your IdP and browser.

Another issue that might crop up would be the secureOnly flag on the cookie - if you are not running Apache Knox with SSL enabled (shame on you) then this flag must not be set. See the KnoxSSO topology and service for setting that to false.

If authentication seems to be successful but there is no listing rendered. Ensure that there is a principal mapping for the username being asserted to the WebHDFS service. Check the audit log for the username being used and map it to "guest" as necessary.

ENJOY!
====
