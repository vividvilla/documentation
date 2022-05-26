```yaml title="config.yaml"
authenticate_service_url: https://authenticate.localhost.pomerium.io
certificates:
  - cert: /pomerium/cert.pem
    key: /pomerium/key.pem
shared_secret: CHANGEME
cookie_secret: CHANGEME
idp_client_id: CHANGEME
idp_client_secret: CHANGEME
idp_provider: google
routes:
  - from: tcp+https://redis.localhost.pomerium.io:6379
    to: tcp://redis:6379
    policy:
      - allow:
          or:
            - domain:
                is: gmail.com

  - from: tcp+https://ssh.localhost.pomerium.io:22
    to: tcp://ssh:2222
    policy:
      - allow:
          or:
            - domain:
                is: gmail.com

  - from: tcp+https://pgsql.localhost.pomerium.io:5432
    to: tcp://pgsql:5432
    policy:
      - allow:
          or:
            - domain:
                is: gmail.com

databroker_storage_type: redis
databroker_storage_connection_string: redis://redis:6379
```