# Idempotency

When writing FSM `Action`s, it is important to write them in such a way that they are 
[idempotent](https://en.wikipedia.org/wiki/Idempotence). During times of system pressure,
downtime etc, actions _WILL_ be executed multiple times. All effort is made in the 
framework to prevent that from happening, but it is impossible to prevent entirely, since 
AWS Lambda is architected for at-least-once delivery of messages.

# So How Do We Do That?

In computer science, the term idempotent is used to describe an operation that will 
produce the same results if executed once or multiple times. Say we want to increment
the value of of a counter in memcache. The following `Action` is NOT idempotent

    class IncrementAction(Action):
      def execute(self, context, obj):
        current_value = memcache.get('counter')
        new_value = current_value + 1
        memcache.set('counter', new_value)
        return 'done'
        
since running it multiple times will result in multiple increments if the failure occurs
anytime after the `memcache.set`.

The following `Action` is also NOT idempotent. Although `memcache.incr `is atomic, multiple 
executions result in multiple increments.

    class IncrementAction(Action):
      def execute(self, context, obj):
        new_value = memcache.incr('counter')
        return 'done'
        
To achieve an idempotent increment, do something like this (does not handle new 
counters, memcache failures,  etc) :

    class IncrementAction(Action):
      def execute(self, context, obj):
        # unique id for the action
        idempotency_flag = context['guid'] + '-increment'
        if not memcache.get(idempotency_flag):
          new_value = memcache.incr('counter')
          # set the idempotency flag so this code won't execute again
          memcache.set(idempotency_flag, True)
        return 'done'
        
The above code is basically what the framework does to make a best effort to avoid
re-running code that it knows has executed already. This approach is not perfect,
and a failure AFTER `memcache.incr` and BEFORE `memcache.set` would result in a
double increment.

To truly achieve idempotency, it is probably necessary to split the action into
multiple actions, at the expense of more messages. This level of granularity is
probably overkill for most processes.

    class CurrentValueAction(Action):
      def execute(self, context, obj):
        context['counter'] = memcache.get('counter')
        return 'done'
        
    class IncrementAction(Action):
      def execute(self, context, obj):
        memcache.set('counter', context['counter'] + 1)
        return 'done'
        
`CurrentValueAction` has no side-effects so is idempotent. `IncrementAction`
uses the value from the context which comes from the AWS Kinesis log and never
changes on subsequent retries.

If you have a situation where multiple machines are mutating the SAME data,
you will need to establish requirements about expected behaviour, since 
`CurrentValueAction` has no side-effects, but can return different values each
time if there are multiple writers.
    

          
        
        