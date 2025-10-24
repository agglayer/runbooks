# Upgrade op-succinct from v3.2 to v3.3

## Run DB Upgrade

Open a connection to the proposer DB server and run the following SQL command:

```
update _sqlx_migrations set checksum='\xd6c22d9a7bb3b2090397ad9693a0caa13e897a92cb0f1eb05be9dad93a695848590510c0ebe1aeb8eb308f671b32537a' where version=2;
```
