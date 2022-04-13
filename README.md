# ttrss-auth-freeipa

*-JMJ-*

FreeIPA authentication plugin for Tiny Tiny RSS

## What is this?
This plugin authenticates users within the local FreeIPA domain.

It provides both "Single Sign-On" capability through integration with a
GSSAPI/Kerberos-enabled webserver, as well as standard authentication using
LDAP binds against the domain's LDAP servers.

This plugin requires php-ldap compiled with SASL support. Check phpinfo() to
verify you have a compatible version. In addition, the PHP process needs
access to kerberos credentials in order to perform LDAP queries
(see [below](#apache-configuration)).

I've only tested this on Rocky Linux 8 with Apache 2.4 and PHP 7.4.

Also, I've never written PHP before. Caveat emptor.

## Configuration Options

The following configuration parameters are supported in `config.php`:

    /*
     * These parameters are optional. If unspecified, autodiscovery will be used.
     */
    putenv('TTRSS_AUTH_FREEIPA_DOMIN=ipa.example.com');
    putenv('TTRSS_AUTH_FREEIPA_REALM=IPA.EXAMPLE.COM');
    putenv('TTRSS_AUTH_FREEIPA_LDAP_URI=ldap://freeipa1.ipa.example.com');
    putenv('TTRSS_AUTH_FREEIPA_BASEDN=dc=ipa,dc=example,dc=com');

    /*
     * If specified, access is only granted to members of at least one of the provided
     * groups. Takes a list of group names.
     */
    putenv('TTRSS_AUTH_FREEIPA_ALLOW_GROUPS=ttrss_users,rss_fans');

    /*
     * If specified, admin privileges are granted to members of at least one of the
     * provided groups. Takes a list of group names. Changes are only applied on login.
     */
    putenv('TTRSS_AUTH_FREEIPA_ADMIN_GROUPS=ttrss_admins,sysadmins');

## Apache Configuration

The following apache configuration provides SSO for the TT-RSS web login endpoint,
and standard authentication for everything else:

    <LocationMatch "^/(index.php)?$">
      <If "%{QUERY_STRING} != 'noext=1'">
        AuthType GSSAPI
        AuthName "FreeIPA Single Sign-On"
        Require valid-user
        # if no kerberos ticket, redirect to TT-RSS login page
        ErrorDocument 401 /index.php?noext=1
      </If>
    </LocationMatch>

Note that performing a GSSAPI negotiation for every single HTTP request is extremely
slow, so you want to limit it to the login page only.

Apache needs a keytab for `HTTP/ttrss.example.com`, and PHP needs a kerberos ticket
to perform LDAP queries. The following `gssproxy.conf` snippet is sufficient (this
also works for kerberized postgres queries):

    [service/ttrss]
    mechs = krb5
    cred_store = client_keytab:/var/lib/gssproxy/clients/ttrss.keytab
    euid = apache
    program = /usr/sbin/php-fpm

    [service/HTTP]
    mechs = krb5
    cred_store = keytab:/var/lib/gssproxy/clients/httpd.keytab
    euid = apache
    program = /usr/sbin/httpd

Be sure to export `GSS_AUTH_PROXY=yes` for your httpd and php-fpm daemons:

    # /etc/systemd/system/httpd.service.d/override.conf
    [Service]
    Environment=GSS_USE_PROXY=yes

    # /etc/php-fpm.d/www.conf
    env[GSS_USE_PROXY] = yes

If you're not using gssproxy, you'll need the usual `KRB5_KTNAME` and
`KRB5_CLIENT_KTNAME` with appropriate permissions.

You'll also need the following if SELinux is enabled:

    $ setsebool -P httpd_can_connect_ldap on
