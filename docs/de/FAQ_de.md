# FAQ

---

:construction:
__THIS PAGE IS BEING TRANSLATED__
:construction:

:construction_worker: Contributors are invited. Please read [CONTRIBUTING](../../CONTRIBUTING.md) and [CONTRIBUTING TRANSLATION](../CONTRIBUTING_TRANSLATION.md) guidelines.

:vulcan_salute: Please remove this note once you're done translating.

---


[![FAQ_cn](https://github.com/yammadev/flag-icons/blob/master/png/CN.png?raw=true)](../cn/FAQ_cn.md)
[![FAQ_de](https://github.com/yammadev/flag-icons/blob/master/png/DE.png?raw=true)](../de/FAQ_de.md)
[![FAQ_en](https://github.com/yammadev/flag-icons/blob/master/png/GB.png?raw=true)](../en/FAQ_en.md)
[![FAQ_id](https://github.com/yammadev/flag-icons/blob/master/png/ID.png?raw=true)](../id/FAQ_id.md)
[![FAQ_pl](https://github.com/yammadev/flag-icons/blob/master/png/PL.png?raw=true)](../pl/FAQ_pl.md)

[About](About_de.md) | [Tutorial](Tutorial_de.md) | [Rule Engine](RuleEngine_de.md) | [GRL](GRL_de.md) | [GRL JSON](GRL_JSON_de.md) | [RETE Algorithm](RETE_de.md) | [Functions](Function_de.md) | [FAQ](FAQ_de.md) | [Benchmark](Benchmarking_de.md)

---

## 1. Grule Panicked on Maximum Cycle

**Question**: I got the following panic message when Grule engine is executed.

```Shell
panic: GruleEngine successfully selected rule candidate for execution after 5000 cycles, this could possibly caused by rule entry(s) that keep added into execution pool but when executed it does not change any data in context. Please evaluate your rule entries "When" and "Then" scope. You can adjust the maximum cycle using GruleEngine.MaxCycle variable.
```

**Answer**: This error indicates a potential problem with the rules you're
having the engine evaluate. Grule continues to execute the RETE network on the
working memory until there are no actions left to execute in the conflict set,
which we will call the natural terminal state.  If your set of rules never
allow the network to reach that terminal state then it would run forever.  The
default configuration for `GruleEngine.MaxCycle` is `5000`, which is what is
used to protect from an infinite cycle of runs in a non-terminal rule set.

You can increase this value if you think your system of rules needs more cycles
in order to terminate, but if you do not believe that is the case, then you
probably have a non-terminating rule set.

Consider this fact:

```go
type Fact struct {
   Payment int
   Cashback int
}
```

And the following rules are defined:

```Shell
rule GiveCashback "Give cashback if payment is above 100" {
    When 
         F.Payment > 100
    Then
         F.Cashback = 10;
}

rule LogCashback "Emit log if cashback is given" {
    When 
         F.Cashback > 5
    Then
         Log("Cashback given :" + F.Cashback);
}
```

Executing these rules on the following fact instance...

```go
&Fact {
     Payment: 500,
}
```

... never terminates. 

```
Cycle 1: Execute "GiveCashback" .... because when F.Payment > 100 is a valid condition
Cycle 2: Execute "GiveCashback" .... because when F.Payment > 100 is a valid condition
Cycle 3: Execute "GiveCashback" .... because when F.Payment > 100 is a valid condition
...
Cycle 5000: Execute "GiveCashback" .... because when F.Payment > 100 is still a valid condition
panic
```

Grule executes the same rule again and again because the **WHEN** condition
continues to yield a valid result.

One way to solve this problem is to change the "GiveCashback" rule to something
like:

```Shell
rule GiveCashback "Give cashback if payment is above 100" {
    When 
         F.Payment > 100 &&
         F.Cashback == 0
    Then
         F.Cashback = 10;
}
```

This definition of the `GiveCashback` rule takes the changing state into
account.  Initially the `Cashback` member will be `0` but because the action
modifies that state, it will fail to match in the next cycle and the terminal
state will be reached.

The above method is somewhat "natural" in that it is the rule conditions that
govern the termination. However, if you cannot terminate the execution in this
natural manner, it is possible to modify the engine's state in the action using
the following:

```Shell
rule GiveCashback "Give cashback if payment is above 100" {
    When 
         F.Payment > 100
    Then
         F.Cashback = 10;
         Retract("GiveCashback");
}
```

The `Retract` function removes the "GiveCashback" rule from the knowledge base
for the next cycle. Since it's no longer present, it cannot be re-evaluated in
that next run. Be aware, though that this only happens for the cycle immediately
following the `Retract` call.  The subsequent cycle will re-introduce the call.

---

## 2. Saving Rule Entry to database

**Question**: Is there a plan to integrate Grule with a database storage system?

**Answer**: No. While it is a good idea to store your rule entries in some sort
of database, Grule will not create any database adapter to automaticaly store
and retrieve rules.  You can easily create such adapter yourself using the
common interfaces on the Knowledgebase: *Reader*, *File*, *Byte Array*, *String*
and *Git*. Strings can be easily inserted and selected from database, as you
load them into Grule's knowledgebase. 

We don't want to couple Grule to any particular database implementation.

---

## 3. Maximum number of rule in one knowledge-base

**Question**: How many rule entry can be inserted into knowledgebase?

**Answer**: You can have as many rule entries you want but there should be at
least one minimum.

---

## 4. Fetch all rules valid for a given fact

**Question**: How can I test my rules for validity against given Facts?

**Answer**: You can use the `engine.FetchMatchingRule` function. Refer this
[Matching Rules Doc](MatchingRules_de.md) for more info

---

## 5. Rule Engine use-case

**Question**: I have read about the rule engine, but what real benefit it can bring? Give us some use-cases.

**Answer**: The following cases are better solved with a rule-engine in my humble opinion.

1. An expert system that must evaluate facts to provide some sort of real-world
   conclusion. If not using a a RETE-style rule engine, one would code up a
   cascading set of `if`/`else` statements and the permutations of the
   combinations of how those might be evaluated would quickly become impossible
   to manage.  A table-based rule engine might suffice but it is still more
   brittle against change and is not terribly easy to code. A system like Grule
   allows you to describe the rules and facts of your system, releasing you from
   the need to describe how the rules are evaluated against those facts, hiding
   the bulk of that complexity from you.

2. A rating system. For example, a bank system may want to create a "score" for
   each customer based on the customer's transaction records (facts).  We could
   see their score change based on how often they interact with the bank, how
   much money they transfer in and out, how quickly they pay their bills, how
   much interest they accrue earn for themselves or for the bank, and so on. A
   rule engine is provided by the developer and the specification of the facts
   and rules can then be supplied by subject matter experts in the bank's
   customer business. Decoupling these different teams puts the responsbilities
   where they should be.

3. Computer games. Player status, rewards, penalties, damage, scores and
   probability systems are many different examples of where rule play a
   significant part of nearly all computer games. These rules can interact in
   very complex ways, often times in ways that the developer didn't imagine.
   Coding these dynamic situations in a scripting language (e.g. LUA) can get
   quite complex, and a rule engine can help simplify the work tremendously.

4. Classification systems. This is actually a generalization of the rating
   system described above.  Using a rule engine, we can classify things such as
   credit eligibility, bio chemical identification, risk assessment for
   insurance products, potential security threats, and many more.

5. Advice/Suggestion system. A "rule" is simply another kind of data, which
   makes it a prime candidate for definition by another program.  This program
   can be another expert system or artificial intelligence.  Rules can be
   manipulated by another system in order to deal with new types of facts or
   newly discovered information about the domain which the rule set is intending
   to model.

There are so many other use-cases that would benefit from the use of
Rule-Engine. The above cases represent only a small number of the potential. 

However it is important to state that a Rule-Engine not a silver bullet, of
course.  Many alternatives exist to solve "knowledge" problems in software and
those should be employed when they are most appropriate. One would not employ a
rule engine where a simple `if` / `else` branch would suffice, for instance.

Theres's someting else to note: some rule engine implementations are extremely
expensive yet many businesses gain so much value from them that the cost of
running them is easily offset by that value.  For even moderately complex use
cases, the benefit of a strong rule engine that can decouple teams and tame
business complexity seems to be quite clear.

---

## 6. Logging

**Question**: Grule's logs are extremely verbose.  Can I turn off Grule's logger?

**Answer**: Yes. You can reduce (or completely stop) Grule's logging by increasing it's log level.

```go
import (
    "github.com/hyperjumptech/grule-rule-engine/logger"
    "github.com/sirupsen/logrus"
)
...
...
logger.SetLogLevel(logrus.PanicLevel)
```

This will set Grule's log to `Panic` level, where it will only emits log when it panicked.

Of course, modifying the log level reduces your ability to debug the system so
we suggest that a higher log level setting only be instituted in production
environments.


---

**Question**: I've just upgraded Grule to the newest version, suddenly no log comes out of Grule?

**Answer**: Yes, as of Grule v1.20.1 a new NoopLogger were introduced. This provide a plain and neutral logging framework Grule.
This allows grule to use logging framework of your choice, what ever it is. Here is how:

1. Create your own version of Logger by implementing "logger.Logger" interface, in this example lets call it `MyLogger`.

```go
type MyLogger struct {}
func (myLog *MyLogger) Debug(args ...interface{}) {
	.. code to add debug log here ..
}
Info(args ...interface{}){
.. code to add info log here ..
}
Warn(args ...interface{}){
.. code to add warn log here ..
}
... and so on
```

2. Instantiate `MyLogger` and add this to `logger.LogEntry` with its default LogLevel (`logger.Level`),

```go
myLogEntry := &LogEntry {
	Logger : &MyLogger{},
	Level : DebugLevel,
}
```

3. Set `logger.Log` to use your new `LogEntry`

```go
logger.Log := myLogEntry
```

If you're already uses `Logrus` or `ZapLog` or `ZeroLog`, you could straightly uses
`logger.SetLogger()` function and Grule will use your logger straight away. 