# Scripting addon for asterisk app_queue

Asterisk app_queue is a very powerfull application for a lot of standard scenario. Its routing design is, however, somewhat limited. 

This patch adds the possibility to influence the queue routing by writing a lua script. We did choose lua because asterisk already depends on it and it is, for a scripting language, really fast.

# Credits and license

See our pascom pbx at http://www.pascom.net for a fully integrated asterisk solution with skill based routing.
This code is licensed under the Gnu General Public Licence (GPL V2). See License File.

# How to build it

Create a patch from the provided modified app_queue__VERSION.c and the app_queue__VERSION__ORIG.c (original src from Digium).

    diff $src/app_queue__VERSION__ORIG.c $src/app_queue__VERSION.c | patch /your/build/folder/asterisk/apps/app_queue.c -

**Check git branches to find patches for other asterisk versions.**

Compile asterisk as usual. The patch was created against asterisk 13.25.0 and only tested with this version. 

# Development

Feel free to enhance the patch with more callbacks or parameters/mapped functions. We will happily accept merge requests for this repository.

# How to use it

## Enable scripting for your queue(s) 

Each queue can have a different routing script.

Add a "luascript" key to your queues.conf and point it to your script. Just put the script into your /etc/asterisk folder or whatever you use as AST_CONFIG_DIR.

Just omit the "luascript" entry and you'll get a plain old app_queue.


## Script callbacks

Asterisk will jump into your routing script if you add specific functions to it. It is not necessary to add all the callbacks, just add what you need. 

All parameters are copied to the lua functions. You do not get direct access to internal asterisk structures by pointers or other means.  

### init

Called on a queue_reload or asterisk startup.  For now the "queue" table only contains the "name" Attribute.
No return value is expected.

```lua
function init(queue) {
   ast_verbose("**** INIT QUEUE " .. queue.name)
}
```

### cleanup

Called on a queue_reload, if your queue is removed or on asterisk shutdown. 
No return value is expected.

```lua
function cleanup() {
  -- close resources...
}
```

### enter_queue

Like the method name imposes: a new waiter enters the queue.
No return value is expected.

```lua
function enter_queue(entry, vars) {
  -- entry is a table with some parameters of the queue_ent structure.
  -- "context","digits","prio","channel","uniqueid","queuename"

  -- vars is a table with all channel variables of the enqueued entry
}
```

### is_our_turn

What if you want to implement a fair queuing or vip routing for entries?

Is_our_turn is polled every 1 second for each waiting call in order to get a answer to the question "do i have to wait any longer?"

Asterisk has here two default modes: 
a) if auto_fill is enabled: distribute the top N calls if N Agents are available
b) auto_fill disabled: distribute the top of the queue if one Agent is available.

Return value:
- -1 let asterisk decide. Same if the callback is not implemented.
- 0 Wait longer
- 1 Do not wait longer, try to distribute the call now.
- 2 Expire this entry immediately. The channel leaves the queue and a QUEUE_TIMEOUT is signalled. 

```lua
function is_our_turn(entry) {
  -- entry is a table with some parameters of the queue_ent structure.
  -- "context","digits","prio","channel","uniqueid","queuename","pending","pos","start","expire","variables"
  -- entry.variables is a table with all channel variables of the enqueued entry
  
  return -1
}
```


### calc_metric

The workhorse of the routing script. Calculate the metrics for a given waiter and agent.

Return value:
- -1 ignore this agent for now.
- 0 do not ignore this agent, use the configured strategy to actually calc the metric
- positive value: absolute metric for this agent. This skips the asterisk strategy. The agent with the lowest value will get the call.

You can use this callback to just filter out agents (return -1, 0) or directly calculate the final metrics.

```lua
function calc_metric(member, entry, vars) {
  -- member is a table with some values from the queues "member" structure.
  -- "interface","membername","queuepos","penalty","paused","calls","dynamic","status"

  -- entry is a table with some parameters of the queue_ent structure.
  -- "context","digits","prio","channel","uniqueid","queuename","pending","pos","start","expire","variables"
  -- entry.variables is a table with all channel variables of the enqueued entry
  
  -- vars is a table with all channel variables of the enqueued entry

  -- zero means: consider this agent but use standard asterisk strategy for calling
  return 0
}
```

## Mapped methods

You can use some asterisk functions directly in the lua script. Also we add some convenience functions.

### Output a verbose message in the cli

- ast_verbose("Hello Asterisk")

### Logging

- ast_log(level, message) 
-- level 0: DEBUG
-- level 1: WARNING
-- level 2: NOTICE
-- level 3: WARNING
-- level 4: ERROR
-- level 5: VERBOSE

### Write something into the queuelog

- ast_queue_log("myqueue","mobydick-1443625288.533","NONE","CUSTOM","LOGENTRY")

### Get one member 

-- get_member(interface)

See calc_metric, parameter "member".

### Get all members as table

- get_members()

You'll get a table with all members in the current queue. Each entry equals to the definition in calc_metric, parameter "member".

### Get all entries as a table

- get_entries()
 
You'll get a table with all waiting calls in the current queue. See calc_metric, parameter "entry" for an explanation.

