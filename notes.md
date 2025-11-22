## Configuring DB Connection Pool

A sql.DB pool contains two types of connections:
- 'in-use' (actively executing an SQL statement)
- 'idle'

When Go code is instructed to perform a db task, it checks if any idle connections area available. If one is available, it reuses it and marks it as 'in-use', otherwise it creates a new additional connection. When Go resuses an idle connection, any problems with it are handled gracefully and a new connection is created.

### Configuring the Pool

4 methods can be used to configure the behavior.

#### SetMaxOpenConns method
SetMaxOpenConns() sets an upper MaxOpenConns limit on the number of 'open' (in-use + idle) connections -- default is unlimited but postgres can't handle more than 100 before erroring.

A limit of below 100 should be set to avoid the error and act as a rudimentary throttle. Limiting the connections could lead to client requests to hang. YOU NEED TO USE THE context.Context object TO SET TIMEOUT!

#### The SetMaxIdleConns method

Increasing number of idle connections (default is 2) helps performance in theory since Go doesn't have to create new connections BUT having idle connections open comes at a memory cost, taking away memory from app code and db execution.

#### The SetConnMaxLifetime method

Longer lifetime in theory means better performance but certain scenarioes, a shorter time is better:
- If your SQL database enforces a maximum lifetime on connections, it makes sense to set
ConnMaxLifetime to a slightly shorter value.
- To help facilitate swapping databases gracefully behind a load balancer.

#### SetConnMaxIdleTime 

Setting is useful because we can set a relatively high limit on number of idle connections in the pool, but we can periodically free up the resources by removing the idle connections that don't end up being used (like after a spike).

### Putting it into practice

So that’s a lot of information to take in… and what does it mean in practice? Let’s
summarize all the above into some actionable points.

1. As a rule of thumb, you should explicitly set a MaxOpenConns value. This should be
comfortably below any hard limits on the number of connections imposed by your database and infrastructure, and you may also want to consider keeping it fairly low to
act as a rudimentary throttle. For this project we’ll set a MaxOpenConns limit of 25 connections. I’ve found this to be a reasonable starting point for small-to-medium web applications and APIs, but ideally you should tweak this value for your hardware depending on the results of benchmarking and load-testing.

2. In general, higher MaxOpenConns and MaxIdleConns values will lead to better
performance. But the returns are diminishing, and you should be aware that having a
too-large idle connection pool (with connections that are not frequently re-used) can
actually lead to reduced performance and unnecessary resource consumption.
Because MaxIdleConns should always be less than or equal to MaxOpenConns, we’ll also
limit MaxIdleConns to 25 connections for this project.

3. To mitigate the risk from point 2 above, you should generally set a ConnMaxIdleTime
value to remove idle connections that haven’t been used for a long time. In this project
we’ll set a ConnMaxIdleTime duration of 15 minutes.

4. It’s probably OK to leave ConnMaxLifetime as unlimited, unless your database imposes a hard limit on connection lifetime, or you need it specifically to facilitate something like gracefully swapping databases. Neither of those things apply in this project, so we’ll leave this as the default unlimited setting.
