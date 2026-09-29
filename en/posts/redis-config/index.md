# Redis Transactions and Configuration


# Transaction

The essence of a Redis transaction is a collection of commands where all commands within the transaction are serialized and executed sequentially. The characteristics include atomicity, sequentiality, and exclusivity.
Redis transactions do not have the concept of isolation levels. All commands in the transaction are not directly executed; they are only executed when the execution command is issued.
While individual Redis commands maintain atomicity, a transaction does not guarantee atomicity.

## Execution of a Redis transaction

Start the transaction (using MULTI).
Queue the commands.
Execute the transaction (using EXEC).
The command to discard the transaction is DISCARD.

## Transaction exceptions in Redis

Compilation Exception (Syntax Error in Code or Command): If there is a syntax error in any command within the transaction, none of the commands in the transaction will be executed.
Runtime Exception: If there is a runtime error in the transaction queue, the commands that are error-free will be executed normally when the transaction is executed, while the erroneous command will throw an exception.

## Locks

### Pessimistic locks

| Pessimistic Lock: This approach assumes that issues may occur at any time, so it locks the resource whenever an operation is performed to prevent conflicts.

### Optimistic locks

| Optimistic Lock: This approach assumes that conflicts or issues are unlikely to happen. It doesn't lock the resource during the operation. Instead, it checks if the data has been modified by someone else before committing the update.
| - Getting the Version: Before updating data, the version of the data is retrieved.
| - Comparing the Version: During the update, the version is compared to ensure no one else has modified the data in the meantime.

Watch Command: Use WATCH [key] to monitor the object for changes before making updates.
Unlock Command: Use UNWATCH to release the lock and stop monitoring.

```bash
# command 1
127.0.0.1:6379> set money 200
OK
127.0.0.1:6379> set out 0
OK
127.0.0.1:6379> watch money
OK
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> decrby money 10
QUEUED
127.0.0.1:6379> incrby out 10
QUEUED
127.0.0.1:6379> exec
(nil)
# command 2
127.0.0.1:6379> get money
"200"
127.0.0.1:6379> set money 1000
OK
```

1. 127.0.0.1:6379> set money 200

   - This command sets a key money in Redis with a value of 200. The 127.0.0.1:6379 indicates the local address and port connected to the Redis server. The response OK signifies that the operation was successful.

2. 127.0.0.1:6379> set out 0

   - This command sets another key out with a value of 0. Again, OK indicates that the operation was successful.

3. 127.0.0.1:6379> watch money

   - Here, the WATCH command is used to monitor the key money. The WATCH command makes Redis keep an eye on this key before executing the transaction. If the value of money is modified by another client before the transaction is executed, the transaction will be aborted. OK indicates that the key is being successfully watched.

4. 127.0.0.1:6379> MULTI

   - The MULTI command begins a transaction. All commands issued after MULTI and before EXEC are queued up and will be executed atomically when EXEC is called. The response OK confirms that the transaction has started.

5. 127.0.0.1:6379> decrby money 10

   - The DECRBY command decreases the value of money by 10. Since this is inside a transaction, the command is queued, and QUEUED indicates that the command has been successfully added to the transaction queue.

6. 127.0.0.1:6379> incrby out 10

   - The INCRBY command increases the value of out by 10. Like the previous command, QUEUED indicates that it has been added to the transaction queue.

7. 127.0.0.1:6379> exec

   - The EXEC command executes all the commands queued in the transaction. If the money key (which was being watched) was modified by another client after the WATCH command but before EXEC, then the transaction will be aborted, and none of the commands will be executed.
   - In this case, the response (nil) indicates that the money key was indeed modified by another client, causing the transaction to be aborted and no commands to be executed.

In summary, this code attempts to decrease the value of money and increase the value of out within a transaction. However, because the money key was modified by another client before the transaction was executed, the transaction was aborted, and no changes were applied.

# Config

Redis commands are case-insensitive, meaning you can enter commands in uppercase, lowercase, or a combination of both, and they will be interpreted the same way. For example, SET, set, and SeT are all valid and will perform the same operation. This allows flexibility in how commands are written and executed.

{{< image src="redis-config1.webp" >}}

In Redis, it is possible to include other configuration files within a primary configuration file. This allows for better organization and modularity of configuration settings. You can use the include directive in the main Redis configuration file to include other configuration files.
This would include the configuration settings from other-config.conf into your main Redis configuration. You can include multiple files this way by adding multiple include directives.
This feature is useful when you want to separate certain configuration aspects, such as security settings, network settings, or performance tuning parameters, into different files and then bring them all together in the main configuration file.

{{< image src="redis-config2.webp" >}}

## Network

The bind directive specifies the network interface addresses that Redis will listen on. By default, Redis binds to all network interfaces, which could expose the server to the internet if not properly configured. You can limit this exposure by specifying one or more IP addresses.

The protected-mode directive is a security feature introduced to prevent unintended exposure of Redis instances to the internet.
When protected-mode is set to yes (which is the default), Redis will only accept connections from the loopback address (127.0.0.1) unless you have explicitly configured the bind directive or set a password with requirepass.
If Redis detects that it is being accessed from an external IP and protected-mode is enabled, it will refuse connections and log a warning.

```bash
bind 127.0.0.1
protected-mode yes
port 6379
```

## General

The daemonize directive controls whether Redis runs as a daemon. By default, this option is set to no, which means Redis runs in the foreground. If you want Redis to run as a background process, you need to set this option to yes.

When daemonize is set to yes, Redis will start in the background and will write its process ID (PID) to a file, usually specified by the pidfile directive in the configuration file. This allows you to manage the Redis process more easily, for example, by stopping or restarting it using standard tools like kill.

```bash
daemonize yes
pidfile /var/run/redis_6379.pid
# log
# Specify the server verbosity level.
# This can be one of:
# debug (a lot of information, useful for development/testing)
# verbose (many rarely useful info, but not a mess like the debug level)
# notice (moderately verbose, what you want in production probably) 
# warning (only very important / critical messages are logged)
loglevel notice
logfile 
database 16
always-show-logo yes
```

## RDB

These save directives define when Redis should automatically save a snapshot of the dataset to disk. The format is save <seconds> <changes>, where Redis will perform a save if at least <changes> keys have been modified in the last <seconds> seconds.

```bash
#  Save a snapshot if at least one key has been modified in the last 900 seconds (15 minutes).
save 900 1
# Save a snapshot if at least 10 keys have been modified in the last 300 seconds (5 minutes).
save 300 10 
# Save a snapshot if at least 10,000 keys have been modified in the last 60 seconds (1 minute).
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dir ./
```

- stop-writes-on-bgsave-error: If an error occurs during the background saving process (BGSAVE), Redis will stop accepting writes to avoid potential data loss. This setting is a safety mechanism to prevent further modifications to the dataset when a persistence error has occurred.
- rdbcompression: Enables LZF compression for the RDB file. This reduces the size of the snapshot file on disk, but it may increase CPU usage during the save process. It is generally recommended to keep this enabled unless disk space is not a concern and you want to minimize CPU usage.
- rdbchecksum: Enables a checksum at the end of the RDB file. This ensures the integrity of the file, allowing Redis to detect potential corruption. The trade-off is a small performance cost during save and load operations.
- dir: Specifies the directory where the RDB file will be saved. The ./ indicates the current directory where the Redis server was started. You can change this to an absolute path if you want to store the RDB file in a specific location on the filesystem.

## Security

```bash
config set requirepass 123456
auth 123456
maxclients 10000
maxmemory <byte>
maxmemory-policy noeviction
```

- config set requirepass: Sets the password required to authenticate clients connecting to the Redis server. In this example, the password is set to 123456. After setting a password, clients must authenticate using the AUTH command before they can execute any other commands.
- auth: This command is used by clients to authenticate with the Redis server using the password set by requirepass. In this example, the client uses 123456 as the password to authenticate.
- maxclients: Limits the maximum number of clients that can be connected to the Redis server simultaneously. In this example, the maximum is set to 10,000. If the limit is reached, Redis will start rejecting new connections with an error until some clients disconnect.
- maxmempoy: Sets the maximum amount of memory that Redis is allowed to use. The value should be specified in bytes (e.g., maxmemory 2gb for 2 gigabytes). Redis will start applying the configured eviction policy when this memory limit is reached.
- maxmemory-policy: Defines the strategy Redis will use when the memory limit (maxmemory) is reached.
  - noeviction: Default setting. Redis will not evict any keys when the memory limit is reached. Instead, it will return an error to any write commands that would require more memory.
  - allkeys-lru: Redis will evict the least recently used (LRU) keys from all keys to make room for new data.
  - volatile-lru: Redis will evict the least recently used (LRU) keys only from keys that have an expiration set (expire).
  - allkeys-random: Redis will randomly evict keys from all keys to free up memory.
  - volatile-random: Redis will randomly evict keys only from those that have an expiration set (expire).
  - volatile-ttl: Redis will evict keys with the shortest remaining time to live (TTL) from those with an expiration set (expire).

## AOF

```bash
appendonly no
appendfilename "appendonly.aof"
appendfsync always
appendfsync everysec
appendfsync no
```

- appendonly: This setting controls whether Redis uses the Append-Only File (AOF) for persistence. By default, it is set to no, meaning AOF is disabled, and Redis relies on RDB (Redis Database) snapshots for persistence. The RDB method is typically sufficient for most cases because it is faster and consumes less disk space, though it might result in more data loss compared to AOF in the event of a crash.
- appendfilename: Specifies the name of the AOF file. The default name is appendonly.aof. This file logs every write operation received by the server, allowing Redis to rebuild the dataset by replaying the AOF file if it needs to recover.
- appendfsync
  - always: Every write operation is immediately flushed to disk (fsync), ensuring the highest level of data durability but at a significant performance cost. This setting guarantees that no data will be lost, but it slows down the server due to the constant disk I/O operations.
  - everysec: Redis will flush data to disk once every second. This is a good compromise between performance and data safety, as it balances the risk of losing up to one second's worth of data with relatively low impact on performance. This is the recommended setting for most use cases.
  - no: Redis will not explicitly flush data to disk. Instead, the operating system handles the flushing, which is the fastest option but offers no guarantees about when the data will be safely written to disk. This could result in significant data loss if the system crashes before the OS syncs the data.


---

> Author: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/en/posts/redis-config/  

