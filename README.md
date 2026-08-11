# YOURLS LDAP / LDAPS Auth Plugin

This is a fork of [mattv8/yourls-ldap-plugin](https://github.com/mattv8/yourls-ldap-plugin) that adds two features for production LDAPS deployments:

1. **Full `ldaps://` URI support** via a new `LDAPAUTH_LDAP_URI` constant.
2. **Strict TLS certificate verification** (`LDAP_OPT_X_TLS_DEMAND`) for StartTLS, instead of `LDAP_OPT_X_TLS_NEVER`.

The original plugin only supports `ldap_connect(host, port)` and, if `LDAPAUTH_START_TLS` is set, it used `LDAP_OPT_X_TLS_NEVER`. This fork lets you point YOURLS at a real `ldaps://` endpoint and still verifies the server certificate.

## Changes vs. upstream

```diff
--- a/plugin.php
+++ b/plugin.php
@@ -126,7 +126,10 @@ function ldapauth_get_ldap_connection()
 	} else {
-		return ldap_connect(LDAPAUTH_HOST, LDAPAUTH_PORT);
+		if (defined("LDAPAUTH_LDAP_URI") && LDAPAUTH_LDAP_URI) {
+			return ldap_connect(LDAPAUTH_LDAP_URI);
+		}
+		return ldap_connect(LDAPAUTH_HOST, LDAPAUTH_PORT);
 	}
@@
 		ldap_set_option($ldapConnection, LDAP_OPT_PROTOCOL_VERSION, 3);
+		ldap_set_option($ldapConnection, LDAP_OPT_X_TLS_REQUIRE_CERT, LDAP_OPT_X_TLS_DEMAND);
 		if (defined("LDAPAUTH_START_TLS") && LDAPAUTH_START_TLS) {
 			@ldap_start_tls($ldapConnection);
 		}
```

## Installation

1. Download `plugin.php`.
2. Copy the plugin folder into your `user/plugins/` folder for YOURLS.
3. Set up the parameters for the plugin in `user/config.php` (see below).
4. Activate the plugin with the plugin manager in the admin interface.

## Usage

When the plugin is enabled and a user was not successfully authenticated using data specified in `$yourls_user_passwords`, an LDAP authentication attempt will be made. If LDAP authentication is successful, then you will immediately go to the admin interface.

You can also set a privileged account to search the LDAP directory with. This is useful for directories that don't allow anonymous binding. If you define a suitable template, the current user will be used for binding. This is useful for Active Directory / Samba.

Setting the groups settings will check if the user is a member of that group before logging them in and storing their credentials. This check is only performed the first time they authenticate or when their password changes.

By default, the plugin implements a simple cache of LDAP users. As well as reducing requests to the LDAP server, this has the effect of allowing the YOURLS API to work with LDAP users.

## Configuration

These are the available constants for `user/config.php`.

### Connection settings

- `define( 'LDAPAUTH_LDAP_URI', 'ldaps://ldap.domain.com:636' );` (new) **Full LDAP URI**, including protocol and port. Use this for `ldaps://`. If set, it overrides `LDAPAUTH_HOST` / `LDAPAUTH_PORT`.
- `define( 'LDAPAUTH_HOST', 'ldap.domain.com' );` (fallback) LDAP host name or IP. Used only if `LDAPAUTH_LDAP_URI` is not defined.
- `define( 'LDAPAUTH_PORT', 636 );` (fallback) LDAP server port — often `389` for `ldap://` or `636` for `ldaps://`.
- `define( 'LDAPAUTH_BASE', 'dc=domain,dc=com' );` Base DN (location of users).
- `define( 'LDAPAUTH_USERNAME_FIELD', 'uid');` (optional) LDAP field name in which the username is stored.
- `define( 'LDAPAUTH_START_TLS', false );` (optional) Set to `true` to use StartTLS on `ldap://` port 389. Not used when `LDAPAUTH_LDAP_URI` is an `ldaps://` URI.

### TLS / certificate notes

- For `ldaps://`, set `LDAPAUTH_LDAP_URI` to the full URI and keep `LDAPAUTH_START_TLS` as `false`.
- For StartTLS on port 389, leave `LDAPAUTH_LDAP_URI` unset and set `LDAPAUTH_START_TLS` to `true`.
- The CA certificate must be in the system's trust store (e.g. `/etc/ssl/certs`, updated with `update-ca-certificates`, or `SSL_CERT_FILE` / `CURL_CA_BUNDLE` for libldap). This plugin now uses `LDAP_OPT_X_TLS_DEMAND`, so a valid certificate is required.

### Privileged account for the user search

- `define( 'LDAPAUTH_SEARCH_USER', 'cn=your-user,dc=domain,dc=com' );` (optional) Privileged user to search with.
- `define( 'LDAPAUTH_SEARCH_PASS', 'the-pass');` (optional) Privileged user password (only if `LDAPAUTH_SEARCH_USER` is set).

### Template to bind using the current user

- `define( 'LDAPAUTH_BIND_WITH_USER_TEMPLATE', '%s@myad.domain' );` (optional) Use `%s` as the placeholder for the current username.

### Custom LDAP search filter

- `define( 'LDAPAUTH_SEARCH_FILTER', '(&(samaccountname=%s)(memberof=YOURLS-ADMINS,OU=Groups,DC=example,DC=com))' );` Use `%s` as the placeholder for the current username. If this option is not set, the filter is based only on `LDAPAUTH_USERNAME_FIELD`.

For AD nested groups, the `LDAP_MATCHING_RULE_IN_CHAIN` OID (1.2.840.113556.1.4.1941) can be used:
```php
define( 'LDAPAUTH_SEARCH_FILTER', '(&(samaccountname=%s)(memberof:1.2.840.113556.1.4.1941:=YOURLS-ADMINS,OU=Groups,DC=example,DC=com))' );
```

### Group membership check

**Attribute based** (default):
- `define( 'LDAPAUTH_GROUP_MODE', 'attribute');`
- `define( 'LDAPAUTH_GROUP_ATTR', 'memberof' );` (optional)
- `define( 'LDAPAUTH_GROUP_REQ', 'the-group;another-admin-group');` Group/s the user must be in. Allows multiple, semicolon-delimited.

**Group based**:
- `define( 'LDAPAUTH_GROUP_MODE', 'group');`
- `define( 'LDAPAUTH_GROUP_BASE', 'ou=groups,dc=example,dc=org' );`
- `define( 'LDAPAUTH_GROUP_ATTR', 'cn' );`
- `define( 'LDAPAUTH_GROUP_MEMBER', 'member' );`
- `define( 'LDAPAUTH_GROUP_MEMBER_TYPE', 'dn' );` either `dn` (default) or `uid`
- `define( 'LDAPAUTH_GROUP_REQ', 'yourlsadmin');` Group/s user must be in. Semicolon-delimited.

Group-based authorization **requires** `LDAPAUTH_SEARCH_USER` and `LDAPAUTH_SEARCH_PASS`.

### Other settings

- `define( 'LDAPAUTH_GROUP_SCOP', 'sub' );` Scope of group search: `sub` (default, checks all the subtree) or `base` (only the exact group).
- `define( 'LDAPAUTH_USERCACHE_TYPE', 1);` `1` caches users in the options table (default). `0` turns off caching.
- `define( 'LDAPAUTH_ADD_NEW', true );` (optional) Add LDAP users to `config.php`.
- `define( 'LDAPAUTH_DNS_SITES_AND_SERVICES', '_ldap._tcp.corporate._sites.yourdomain.com' );` If using AD with multiple DCs, this DNS SRV lookup is used to find active LDAP server names. If set, it overrides the hostname portion of `LDAPAUTH_HOST`.
- `define( 'LDAPAUTH_LDAP_OPT_REFERRALS', 0 );` (optional) Defaults to `1`. Set to `0` to prevent following AD referrals, which can sometimes cause `ldap_search` to hang.

## Example: full `ldaps://` configuration

```php
define( 'LDAPAUTH_LDAP_URI', 'ldaps://ad.example.com:636' );
define( 'LDAPAUTH_BASE', 'DC=example,DC=com' );
define( 'LDAPAUTH_USERNAME_FIELD', 'sAMAccountName' );
define( 'LDAPAUTH_SEARCH_USER', 'CN=ldap,DC=example,DC=com' );
define( 'LDAPAUTH_SEARCH_PASS', 'your-secure-password' );
define( 'LDAPAUTH_START_TLS', false );
define( 'LDAPAUTH_ALL_USERS_ADMIN', true );
define( 'LDAPAUTH_USERCACHE_TYPE', 1 );
```

## Example: StartTLS on port 389

```php
define( 'LDAPAUTH_HOST', 'ad.example.com' );
define( 'LDAPAUTH_PORT', 389 );
define( 'LDAPAUTH_BASE', 'DC=example,DC=com' );
define( 'LDAPAUTH_USERNAME_FIELD', 'sAMAccountName' );
define( 'LDAPAUTH_SEARCH_USER', 'CN=ldap,DC=example,DC=com' );
define( 'LDAPAUTH_SEARCH_PASS', 'your-secure-password' );
define( 'LDAPAUTH_START_TLS', true );
define( 'LDAPAUTH_ALL_USERS_ADMIN', true );
define( 'LDAPAUTH_USERCACHE_TYPE', 1 );
```

## Troubleshooting

- Check the PHP error log, usually at `/var/log/php.log`.
- Check your web server logs.
- You can enable `LDAPAUTH_DEBUG` to print more debug info.

## About the user cache

When a successful login is made against an LDAP server, the plugin will cache the username and encrypted password. Currently, this is done by saving them in an array in the YOURLS options table. This has some advantages:

- It reduces requests to the LDAP server.
- It means that users can still log in even if the LDAP server is unreachable.
- It means that the YOURLS API can be used by LDAP users.

Unfortunately, the cache will not scale well. This is because it integrates tightly with YOURLS's internal auth mechanism, and that does not scale. If you have a few tens of LDAP users likely to use your YOURLS installation, it should be fine. Much more than that, and you may see performance issues. If so, you should probably disable the cache. This will mean that your LDAP users will not be able to use the API. At least not unless they are also listed in `users/config.php`, which suffers from the same scaling problems.

## License

Original Plugin Author(s):  
Copyright 2013 K3A, #1davoaust  
Copyright 2013 Nicholas Waller (code@nicwaller.com) as I used some parts of his CAS authentication plugin :)

Maintainer(s):  
Matt Visnovsky #mattv8

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.
