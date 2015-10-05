# Scripting addon for asterisk app_queue

Asterisk app_queue is a very powerfull application for a lot of standard scenario. Its routing design is, however, somewhat limited. 

This patch adds the possibility to influence the queue routing by writing a lua script. We did choose lua because asterisk already depends on it and it is, for a scripting language, really fast.

# Credits and license

See our mobydick pbx at http://www.pascom.net for a fully integrated asterisk solution with skill based routing.
This code is licensed under the Gnu General Public Licence (GPL V2). See License File.

# How to build it

Create a patch from the provided modified app_queue.c and the ORIG_app.queue.c (Asterisk 11.6-cert11)
     diff $src/ORIG_app_queue.c $src/app_queue.c | patch /your/build/folder/asterisk/apps/app_queue.c -

Compile asterisk as usual. The patch was created against asterisk 11.6 and only tested with this version. 

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

´´´lua
function init(queue) {
   ast_verbose("**** INIT QUEUE " .. queue.name)
}
´´´

### cleanup

Called on a queue_reload, if your queue is removed or on asterisk shutdown. 
No return value is expected.

´´´lua
function cleanup() {
  -- close resources...
}
´´´

### enter_queue

Like the method name imposes: a new waiter enters the queue.
No return value is expected.

´´´lua
function enter_queue(entry, vars) {
  -- entry is a table with some parameters of the queue_ent structure.
  -- "context","digits","prio","channel","uniqueid","queuename"

  -- vars is a table with all channel variables of the enqueued entry
}
´´´

### calc_metric

The workhorse of the routing script. Calculate the metrics for a given waiter and agent.

Return value:
- -1 ignore this agent for now.
- 0 do not ignore this agent, use the configured strategy to actually calc the metric
- positive value: absolute metric for this agent. This skips the asterisk strategy. The agent with the lowest value will get the call.

You can use this callback to just filter out agents (return -1, 0) or directly calculate the final metrics.

´´´lua
function calc_metric(member, entry, vars) {
  -- member is a table with some values from the queues "member" structure.
  -- "interface","membername","queuepos","penalty","paused","calls","dynamic","status"

  -- entry is a table with some parameters of the queue_ent structure.
  -- "context","digits","prio","channel","uniqueid","queuename"

  -- vars is a table with all channel variables of the enqueued entry

  -- zero means: consider this agent but use standard asterisk strategy for calling
  return 0
}
´´´

## Mapped asterisk methods

You can use some asterisk functions directly in the lua script. 

### Output a verbose message

- ast_verbose("Hello Asterisk")

### Write something into the queuelog

- ast_queue_log("myqueue","mobydick-1443625288.533","NONE","CUSTOM","LOGENTRY")

