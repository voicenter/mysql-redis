


# mysql-redis :rocket:

Transform your mysql server with `Redis` caching layer for `mysql/mysql2`.
- MysqlRedis checks if there is a cached result for the query in redis 
- if not found in cache, it will retrieve data from mysql and on successful result cache it in redis for future queries
- if redis is unavailable or errors, query will be served by mysql

### Hashing 
 The above is achieved by creating a unique hash for every query
  `"select 1+2"` => #️⃣
In redis, the hash and query results are stored as key-value pair
#️⃣=> `[ { '1+2': 3 } ]`

#### Currently supported hash types are: 
  
**farmhash32** ⚡🗜️
  Example redis key: `prefix.2jNDCJ`
  Fast!!! Over ~5 million hashes/s on reference machine most compact key.
  
 **farmhash64** 
  Example redis key:  `prefix.DiHlF3yv0V$` 
  fast (~2 million hashes/sec on reference machine )
  *farmhash32/64* use Google's farmhash, non-crypto algorithm (if you have millions of possible queries, these hashes can collide :collision:) so use it for hundreds or thousands of static queries

**blake2b512** 🛡️
Example redis key: `prefix.4KbMOx3xJi+7mJNy0tDbju6NY9uHqOroDsG4rYjpHK1mEwXJokls5Ofdjs7iDsn3cAtibgUkT8RDdpCE2phhiQ==` 
Crypto safe, ~500k hashes/sec on reference machine
Use it for caching millions of different queries (eg. chats, logs)
Note that the key is longer than farmhash.

**full** 
Matches full query string. Use this if you are paranoid or if your queries are smaller than blake2b512 hashes


### Use-case
Use _along_ with mysql and redis

- **No brainer** for retrieving static data, eg, `select * from countries`
- Useful for data that will not be updated once created, eg, `select message from chat where id=?`
- Do not use/Use with caution where data may get updated in mysql as redis cache may be stale
- Use it only with queries/stored procedures that will only select

## Getting Started

### Pre-Requisites
mysql or mysql2, and redis.

For async/await api, you can use mysql2's promise api and [redis-async](https://www.npmjs.com/package/mysql-redis)

### Installing
`npm i mysql-redis --save` 

### Usage
```
const { MysqlRedis, HashTypes } = require("mysql-redis");

// or if you use async await api
const { MysqlRedisAsync, HashTypes } = require("mysql-redis");
```

####  Creating an instance of MysqlRedis requires 
- a mysql connection or pool (mysqlRedis will call it's query method when no cache found)
- redis connection (mysqlRedis will call its set and get methods)
- cache options (optional)  

####  Creating an instance of MysqlRedisAsync requires 
- a mysql connection or pool promise 
	``` 
	// Example from mysql2 docs:
	const  poolPromise  =  mysql.createPool({host:'localhost', user:  'root', database:  'test'}).promise(); 
	```
- async redis
	```
	eg:
	const  asyncRedis  =  require("async-redis");
	const  redis  =  asyncRedis.createClient(redisOptions);

	```
- cache options (optional)  

```
const cacheOptions = {
    expiry: 2629746,// seconds, defaults to 30 days 
    keyPrefix: "sql." // default
    hashType: HashTypes.farmhash
};

const mysqlRedis = new MysqlRedis(
    mysqlConnection,
    redisConnection,
    cacheOptions
);
```
Now if you wish to get something from cache, just use mysqlRedis.query instead of your mysql connection's query. (Use your mysql connection normally to bypass cache)
```
mysqlRedis.query('select * from logs where id =?",["some-log-id"], (err,data,fields)=>{
	console.log(data)
	// if served by Redis, fields value is something like [ { cacheHit: 'sql.Dh9VSNbN5V$' } ]
	// else mysql fields
});
```

or if you like promises, then:

```
const mysqlRedis = new MysqlRedisAsync(
    mysqlConnection,
    redisConnection,
    cacheOptions
);

...
try{

	[result,fields]=await mysqlRedis.query("select 1+?+?",[2,3]);

}catch(err){
	// handle err
}

```
You can override cache options per query as below:

```
mysqlRedis.query('select * from logs where id =?",["some-log-id"],
	{ //cache option
		keyPrefix:'sql-abc-', 
		expire:3600, 
		hashType: HashTypes.farmhash 
	}, 
	(err,data,fields)=>{
	console.log(data)
	// if served by Redis, fields value is something like [ { cacheHit: 'sql.Dh9VSNbN5V$' } ]
	// else mysql fields
});


// promise api
[result,fields]=await mysqlRedis.query("select 1+?+?",[2,3],{ keyPrefix:'your-preferred-prefix-', expire:3600 });

```

 
## Contributing

 Feel free to fork/send PR

## Authors

* **Gi Singh** 

## License

This project is licensed under the [MIT](./LICENSE).

