
##### Configuring your LDAP server is beyond the scope of this document. There are many different approaches and this will most likely be done by IT staff. It's assumed here that you already have a running LDAP or Active Directory server.


There isn't much that you need to do in your application to use LDAP. Just install this plugin, and configure any required parameters and whatever optional parameters you want in Config.groovy. These are described in detail in [Chapter 3|guide:3. Configuration] but typically you only need to set these properties

```
grails.plugin.springsecurity.ldap.context. managerDn = 'uid=admin,ou=system'
grails.plugin.springsecurity.ldap.context. managerPassword = 'secret'
grails.plugin.springsecurity.ldap.context. server = 'ldap://localhost:10389'
grails.plugin.springsecurity.ldap.authorities. groupSearchBase = 'ou=groups,dc=yourcompany,dc=com'
grails.plugin.springsecurity.ldap.search. base = 'dc=yourcompany,dc=com'
```

Often all role information will be stored in LDAP, but if you want to also assign application-specific roles to users in the database, then add this

```
grails.plugin.springsecurity.ldap.authorities.retrieveDatabaseRoles = true
```

to do an extra database lookup after the LDAP lookup.

Depending on how passwords are encrypted in LDAP you may also need to configure the encryption algorithm, e.g.

```
grails.plugin.springsecurity.password.algorithm = 'SHA-256'
```

##  Sample Config.groovy settings for Active Directory

Active directory is somewhat different although still relatively painless if you know what you are doing. Use these example configuration options to get started (tested in Windows Server 2008):

{note}
Replace the placeholders inside [] brackets with appropriate values and remove the [] chars
{note}

```
// LDAP config
grails.plugin.springsecurity.ldap. context.managerDn = '[distinguishedName]'
grails.plugin.springsecurity.ldap. context.managerPassword = '[password]'
grails.plugin.springsecurity.ldap. context.server = 'ldap://[ip]:[port]/'
grails.plugin.springsecurity.ldap. authorities.ignorePartialResultException = true // typically needed for Active Directory
grails.plugin.springsecurity.ldap. search.base = '[the base directory to start the search.  usually something like dc=mycompany,dc=com]'
grails.plugin.springsecurity.ldap. search.filter="sAMAccountName={0}" // for Active Directory you need this
grails.plugin.springsecurity.ldap. search.searchSubtree = true
grails.plugin.springsecurity.ldap. auth.hideUserNotFoundExceptions = false
grails.plugin.springsecurity.ldap. search.attributesToReturn = ['mail', 'displayName'] // extra attributes you want returned; see below for custom classes that access this data
grails.plugin.springsecurity.providerNames = ['ldapAuthProvider', 'anonymousAuthenticationProvider'] // specify this when you want to skip attempting to load from db and only use LDAP

// role-specific LDAP config
grails.plugin.springsecurity.ldap.useRememberMe = false
grails.plugin.springsecurity.ldap.authorities.retrieveGroupRoles = true
grails.plugin.springsecurity.ldap.authorities.groupSearchBase ='[the base directory to start the search.  usually something like dc=mycompany,dc=com]'
// If you don't want to support group membership recursion (groups in groups), then use the following setting
// grails.plugin.springsecurity.ldap.authorities.groupSearchFilter = 'member={0}' // Active Directory specific
// If you wish to support groups with group as members (recursive groups), use the following
grails.plugin.springsecurity.ldap.authorities.groupSearchFilter = '(member:1.2.840.113556.1.4.1941:={0})' // Active Directory specific
```

## Custom UserDetailsContextMapper

There are three options for mapping LDAP attributes to @UserDetails@ data (as specified by the @grails.plugin.springsecurity.ldap.mapper.userDetailsClass@ config attribute) and hopefully one of those will be sufficient for your needs. If not, it's easy to implement [UserDetailsContextMapper](http://static.springsource.org/spring-security/site/docs/3.0.x/apidocs/org/springframework/security/ldap/userdetails/UserDetailsContextMapper.html) yourself.

Create a class in @src/groovy@ or @src/java@ that implements [UserDetailsContextMapper](http://static.springsource.org/spring-security/site/docs/3.0.x/apidocs/org/springframework/security/ldap/userdetails/UserDetailsContextMapper.html) and register it in @grails-app/conf/spring/resources.groovy@:

```
import com.mycompany.myapp.MyUserDetailsContextMapper

beans = {
   ldapUserDetailsMapper(MyUserDetailsContextMapper) {
      // bean attributes
   }
}
```

For example, here's a custom @UserDetailsContextMapper@ that extracts three additional fields from LDAP (fullname, email, and title)

```
package com.mycompany.myapp

import org.springframework.ldap.core.DirContextAdapter
import org.springframework.ldap.core.DirContextOperations
import org.springframework.security.core.userdetails.UserDetails
import org.springframework.security.ldap.userdetails.UserDetailsContextMapper

class MyUserDetailsContextMapper implements UserDetailsContextMapper {

   UserDetails mapUserFromContext(DirContextOperations ctx, String username,
                                  Collection authorities) {

      String fullname = ctx.originalAttrs.attrs['name'].values[0]
      String email = ctx.originalAttrs.attrs['mail'].values[0].toString().toLowerCase()
      def title = ctx.originalAttrs.attrs['title']

      new MyUserDetails(username, null, true, true, true, true,
                        authorities, fullname, email,
                        title == null ? '' : title.values[0])
   }

   void mapUserToContext(UserDetails user, DirContextAdapter ctx) {
      throw new IllegalStateException("Only retrieving data from AD is currently supported")
   }
}
```

and a custom @UserDetails@ class to hold the extra fields:

```
package com.mycompany.myapp

import org.springframework.security.core.GrantedAuthority
import org.springframework.security.core.userdetails.User

class MyUserDetails extends User {

   // extra instance variables
   final String fullname
   final String email
   final String title

   MyUserDetails(String username, String password, boolean enabled, boolean accountNonExpired,
         boolean credentialsNonExpired, boolean accountNonLocked,
         Collection<GrantedAuthority> authorities, String fullname,
         String email, String title) {

      super(username, password, enabled, accountNonExpired, credentialsNonExpired,
            accountNonLocked, authorities)

      this.fullname = fullname
      this.email = email
      this.title = title
   }
}
```

Here we extend the standard Spring Security @User@ class for convenience, but you could also directly implement the interface or use a different base class.
