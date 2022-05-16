# Configuration

Some KC parameters should be applied before running KC through the `${kc.home.dir}/conf/keycloak.conf` properties file:

- Disable HTTP connections (recommended in production mode).

- Set the logging configuration (to know what is happening in the server).

> Where `kc.home.dir` variable is the KC installation directory.

## Disabling HTTP connections

For production environments, you should never expose KC endpoints through HTTP, as it exchanges sensitive data with other applications. So, you have to disable HTTP connections (stop listening HTTP port).

```properties
# Disable http (only for production mode)
http-enabled=false
```

## Logging

By default, KC only writes log information in console, so you have to change this behaviour to also write logs into a file. Next properties allow logging information to a file and set DEBUG level only for `org.keycloak` category:

```properties
# Log 
log=console,file
log-file=${kc.home.dir}/keycloak.log
log-level=INFO,org.keycloak:debug,org.keycloak.transaction:info,org.keycloak.services.scheduled:info
```

## References

- [Configuring logging - Keycloak](https://www.keycloak.org/server/logging)
