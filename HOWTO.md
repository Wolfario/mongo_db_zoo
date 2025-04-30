# Návod ke zprovoznění semestrální práce

## Zapnutí a konfigurace

1. `docker compose up -d`

2. `docker exec -it mongo1 bash`

3. `mongosh`

Následující příkaz slouží k inicializaci replikační sady (replica set). Replikační sada je soubor vzájemně propojených MongoDB serverů, který poskytuje vysokou dostupnost a odolnost proti výpadkům databáze. Celkově tato komanda inicializuje replikační sadu s názvem "rs0" a třemi členy, kteří jsou hostováni na serverech s adresami **mongo1:27017**, **mongo2:27017** a **mongo3:27017**.

4. 
```js
rs.initiate(
  {
    _id: "rs0",
    members: [
      { _id: 0, host: "mongo1:27017", priority: 2 },
      { _id: 1, host: "mongo2:27017", priority: 1 },
      { _id: 2, host: "mongo3:27017", priority: 1 }
    ]
  }
)
```

5. Počkejme chvíli, až se staneme **Primary** (výpis: `rs0 [direct: primary]`)

6. `exit` (x2)

7. `docker cp .\generated_json\MOCK_DATA.json mongo1:/home`

8. `docker exec -it mongo1 bash`

9. `mongosh`

10. `use animals`

11. 
```js
 db.createUser( { user: "test",
                 pwd: "pass",
                 roles: [ { role: "clusterAdmin", db: "admin" },
                          { role: "readAnyDatabase", db: "admin" },
                          "readWrite"] }
)
```

12. `db.auth("test", "pass")`

13. `exit`

14. `mongoimport -u test -p pass -d animals -c animals /home/MOCK_DATA.json`

15. `mongosh`

16. `use animals`

17. `db.auth("test", "pass")`

## Vypnutí

``docker-compose down``