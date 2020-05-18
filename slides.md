# Postgres Progress
## on Temporal Databases

Paul A. Jungwirth<br/>
May 2020<br/>
PGCon 2020

Note:

- Thanks for coming!



# Patches

- Multiranges
- SQL:2011 Temporal
  - Primary Keys
  - Foreign Keys
  - `UPDATE FOR PORTION OF`
  - `DELETE FOR PORTION OF`

Note:

- I've got two patches. Neither is finished yet.
  - Multiranges is close (I hope).
  - I want to talk about what these patches do, and also the approach I've used to implement them.
  - I'm happy for feedback on either of those things!
  - And by the way I want to thank all the people who've contributed time reviewing those patches and offering feedback on the mailing list.

- I gave a talk at PGCon last year about how temporal database work and the SQL:2011 standard.
- This year I'm going to focus more on the work I've done and the implementation side.
  - I'll especially focus on hard parts where someone else may have a better idea.
  - So this talk is really intended for Postgres hackers, although others might find it interesting.
    - I'm kind of a dillettante Postgres hacker.
      - It's not my full-time job, and usually I'm writing Ruby or Python or Javascript, maybe Java.
      - My Postgres work is pretty much all in the evenings.
      - But I've done C off-and-on since high school, and I love getting to work on something a bit more challenging than webapps.
      - I've found Postgres a wonderful community to be a part of.
      - I work with Postgres a lot, both as an application developer and doing DBA/sysadmin/devops things.
        - I've written a bunch of extensions over the years, including a proof-of-concept for temporal primary & foreign keys.
    - So working on these patches has taught me a lot,
      and hopefully I can share some tidbits that might help other nighttime Postgres hackers.
      - There are other talks and articles out there introducing Postgres hacking,
        but there is room for another one, and maybe I'll give one some day.



# Multiranges

![range vs multirange](img/range_vs_multirange.png)

Note:

- Multiranges is a new type. It's a lot like our existing range type, but a little different.
- Here is green we have a range.
  - It has upper and lower bounds.
- In yellow we have a multirange.
  - It's a sequence of non-touching ranges.
  - So unlike a range, it can have gaps.



# Multiranges

<pre><code>[1,3)
empty

{[1,3), [7,9)}
{}

<div class="fragment"><s>{empty}</s>
<s>{[1,4), [4,9), [6,9)}</s></div></code></pre>

Note:

- Here they are as strings.
  - First are a couple ranges,
  - then a couple multiranges
  - Empty and empty
- (next)
- They are automatically canonicalized, so you can't get these.



# Multiranges: Closure

<pre><code>
[1,4) + [7,9)       -- boom
[1,9) - [4,7)       -- boom

<div class="fragment">{[1,4)} + {[7,9)}   -- {[1,4), [7,9)}
{[1,9)} - {[4,7)}   -- {[1,4), [7,9)}</div></code></pre>

Note:

- One thing multiranges add is mathematical closure.
- By closure I don't mean when a function closes over outside variables.
- The math concept of closure is that if you have a domain of values and an operator,
  then the operator always gives a result form the same domain, no matter what inputs you use.
  - So if you take the positive numbers, addition is closed, but not subtraction.
- Ranges are not closed for union or difference (which we write as plus and minus), because both can give gaps.
  - If you do this today Postgres raises an error.
- (next)
- But if you use multiranges, you don't get an error.



# Multiranges

ranges    | multiranges
--------- | --------------------
int4range | int4multirange 
tsrange   | tsmultirange 
...       | ...
textrange | textmultirange
...       | ...
anyrange  | anymultirange 
anycompatiblerange  | anycompatiblemultirange

Note:

- Every built-in range type comes with a multirange type
  - Strictly speaking ranges and multiranges are not types but types of types.
    - There is int4range, daterange, etc.
- Creating a new range type automatically creates a new multirange type
- Here I'm showing a couple built-in ranges,
  a custom range,
  and range polymorphic types.
  - They all have multirange equivalents.



# Multiranges: Memory

```
typedef struct
{
    char    vl_len_[4];       /* varlena header */
    Oid     multirangetypid;  /* multirange type's OID */
    uint32  rangeCount;       /* the number of ranges */

    /* RangeType structs here, which are also varlena... */
} MultirangeType;
```

Note:

- Here's the memory layout of a multirange.
- They are varlena.
  - They have the oid of the concrete multirange type.
  - Next they have a count and then a bunch of ranges.
  - Because range types are also varlena, we can't include them in the struct here.
    - In fact inside a range type, the upper and lower bounds can be varlena too.
    - So ranges have a similar-looking struct that only has the type oid.
  - So when iterating over the ranges you have to advance a pointer by hand.
    - There is a `multirange_deserialize` function that gives you a `RangeType**` to hide those details.

- Btw a lot of the code I'll show has comments omitted and whitespace changed
  so what I want to show will fit on a slide.
  - Don't be alarmed! :-)

- I got a request to store less than the full RangeType in each entry,
  since they all contain the same type oid.
  - You already have the multirange type oid, so you don't need the range type oid at all.
  - But we include it in the range type struct over and over.
  - But only for on-disk possibly.
  - I actually don't know how to have an on-disk format different from in-memory.
    - Maybe someone can help me understand how to do that.
  - But since most access goes through `multirange_deserialize` already,
    I think I could store something smaller and build the `RangeType` list on the fly.
    - I'm skeptical this would actually be a performance win though, compared to the inefficiency of repeating the range type oid.
      - Maybe more experienced hackers can share their judgment about that.



# Multiranges: Memory

```
*ranges = palloc0(*range_count * sizeof(RangeType *));

ptr = (char *) multirange;
end = ptr + VARSIZE(multirange);
ptr = (char *) MAXALIGN(multirange + 1);
i = 0;
while (ptr < end)
{
    r = (RangeType *) ptr;
    (*ranges)[i++] = r;

    ptr += MAXALIGN(VARSIZE(r));
}
```

Note:

- this is from `multirange_deserialize`:
  - it does the pointer hopping to give you a list of ranges.
  - There are alignment issues too because ranges have to be aligned, and so do their upper and lower bounds.



# Multiranges: Operators

```
= <> < <= > >=
<< >> -|-
@> <@ 
&> &<
&&
+ - *
```

Note:

- These are pretty much the same as ranges
- Many accept mixed ranges & multiranges
  - The idea is to make it easy to move between them.
- Nothing too interesting about implementing these.
  - I tried to delegate to range functions wherever possible.



# Multiranges: Identities

```
x + {} = x
x - {} = x
x * {} = {}

x @> y ≡ x + y = x
x @> {}
{} @> x ≡ x = {}

x && y ≡ x * y ≠ {} where x ≠ {} and y ≠ {}
x && {}
```

Note:

- There are lots of mathy properties of ranges that multiranges preserve.
- Ranges and multiranges have nice additive properties:
  - `{}` is an additive identity
- Thinking about ranges mathematically is fun but as far as I can tell there aren't nice multiplicative properties.
  - For example what is the multiplicative identity? If it's (null,null) then you can't find inverses.
  - But at least we can say that x * 0 = 0.

- x includes y is the same as saying that y adds nothing new to x.
- every range includes the empty range.
  - Even the empty range includes the empty range.
  - If the empty range includes something, it must be empty too.

- x overlaps y is the same as saying that x intersects y is not empty (unless x or y are empty already)
- empty overlaps every range (just like it's contained by them)

- Anyway all this is just to say that I'm trying not to *lose* anything from ranges.
- In all these formulas, you can put ranges *or* multiranges for x and y.



# Multiranges: Polymorphic

| poly | concrete |
| ---- | -------- |
| anyelement    | integer
| anyarray      | integer[]
| anyrange      | int4range
| anymultirange | int4multirange |

Note:

- Postgres has this polymorphic type system to let you write generic functions.
  - There is already an anyrange type, and I added an anymultirange type.
- Here are a few of the polymorphic types along with examples of a concrete type.
- The concrete types have to all be compatible.
- If any of these are known, the polymorphic type system can infer the others.
- Except a known element/array doesn't let you infer the range/multirange,
  because you can define multiple range types for the same base type.
- But a known range lets you infer a multirange and vice versa



# Multiranges: Functions

```
lower(anymultirange) returns anyelement
upper(anymultirange) returns anyelement

multirange(r anyrange) returns anymultirange
range_merge(r anymultirange) returns anyrange

range_agg(r anyrange) returns anymultirange
range_intersect_agg(r anyrange) returns anymultirange
```

Note:

- Multiranges comes with a few non-operator functions.
  - `lower` & `upper` return what you'd expect
  - `multirange` is a constructor from a single range.
    - There are variadic type-specific constructors, but it's handy to have one that works for any type.
  - `range_merge` gives the range that includes the whole multirange
    - Sort of the opposite of the constructor
  - `range_agg` is an aggregate function:
    - Take a lot of ranges and combine them. 
  - `range_intersect_agg` is the same but instead of `+` it's `*`.



# Multiranges: Selectivity

<pre><code>{ oid => '8073',
  oid_symbol => 'OID_RANGE_OVERLAPS_MULTIRANGE_OP',
  descr => 'overlaps', oprname => '&&',
  oprleft => 'anyrange', oprright => 'anymultirange',
  oprresult => 'bool',
  oprcom => '&&(anymultirange,anyrange)',
  oprcode => 'range_overlaps_multirange',
  <b>oprrest => 'multirangesel'</b>,
  oprjoin => 'areajoinsel' },
</code></pre>

Note:

- When you use operators in WHERE conditions or JOINs,
  the Postgres planner has to decide how to fit that into the broader query.
  - The biggest question is how many rows you're going to get back.
    - If your condition returns practically the whole table, it's faster to ignore indexes and do a full table scan.
    - We can also make better decisions about how to nest our loops, when to build in-memory bitmaps, etc.
  - So Postgres keeps statistics on the distribution of values in every column,
    and then it calls selectivity functions to estimate what portion of the table a condition will return.
    - Your selectivity functions can consult the statistics and return a number from 0 to 1.
- Still working on this actually
  - Collecting the statistics is done.
  - Computing selectivities is mostly done.
- I added a `multirangesel` function
  - Here I'm showing the details for the overlaps operator.
    - We're telling it to use my multirangesel function to compute selectivity.
- A lot like ranges
- Analysis builds histograms of lower/upper bounds and length
- Selectivity is easy (similar to scalars) for everything but contains/overlaps.
  - Those require both a lower/upper and a length.
  - This is just what rangesel does.
- This will be biased toward denser multiranges: ones without gaps.
  - Probably good to add a "density" histogram we can combine with length
    - Or even redefine length as length times density.

- That's all I have to say about multiranges, so let's talk about temporal features.



# Primary Keys

```
CREATE CONSTRAINT pk_houses
  PRIMARY KEY
  (id, valid_at WITHOUT OVERLAPS);
```

Note:

- This is what you say to define a temporal primary key.
- Id is an ordinary column, like an integer.
  - I'll call this the "scalar" part, maybe betraying some Perl influence.
  - It can be whatever type you like.
  - You can multiple columns.
- `valid_at` is a range column, or hopefully soon a PERIOD.
  - PERIODs are weird SQL:2011 things that are like ranges but not as good.
  - I'd like to support both for compatibility sake, but personally in my own projects I'll just use ranges.
- Of course you can also specify a temporal PK in your CREATE TABLE.



# Primary Keys

![houses primary key](img/houses_pk.png)

Note:

- It's okay if you have a duplicate of the ID, but not if their times overlap.
  - Imagine this is a table of houses.
    - You have two records for house 1, maybe because it was re-appraised.
      - It had one value in 2018 and another value in 2019.



# Primary Keys
<!-- .slide: style="font-size: 95%" -->

```
IndexElem *iparam = makeNode(IndexElem);
iparam->name = pstrdup(without_overlaps_str);
index->indexParams = lappend(index->indexParams, iparam);

opname = list_make1(makeString("&&"));
index->excludeOpNames = lappend(index->excludeOpNames, opname);
index->accessMethod = "gist";
constraint->access_method = "gist";
```

Note:

- Normally PKs are a UNQIUE constraint with an appropriate index.
- Temporal PKs are EXCLUSION constraints with a different kind of index.
- There is already code to include all the scalar parts in the index.
- We add code to include the WITHOUT OVERLAPS part
  - Also use the overlaps operator instead of the equals operator.



# Foreign Keys

```
CREATE CONSTRAINT fk_rooms_to_houses
  FOREIGN KEY
  (house_id, PERIOD valid_at)
  REFERENCES
  (id, PERIOD valid_at);
```

Note:

- Here is the syntax for a temporal foreign key.
  - First you have the scalar part of the key.
    - Again, there can be more than one column.
  - Then you give a range.
    - You need a range from the referencing table and from the referenced table.
- Note that you use `WITHOUT OVERLAPS` for a PK and `PERIOD` for a FK.
  - Also note the `PERIOD` comes before the column name but `WITHOUT OVERLAPS` comes after.
  - I apologize: that's just what SQL:2011 says you should do.
- Of course you can also specify a temporal FK in your CREATE TABLE.



# Foreign Keys

![houses foreign key](img/houses_fk.png)

Note:

- Here I'm showing two table, houses and rooms.
  - Naturally rooms are child records of houses.
    - You can't have a room if you don't have a house.
  - The houses table has a couple green records.
    - They are both for house 1.
  - The rooms table has a yellow record and a red record.
    - Room 1 and room 2.
    - Suppose they both reference house 1.
- The FK value has to be present in the referenced table *for the whole time*
  - The red row is invalid: the room can't exist before the house does.
- It might take more than one row in the PK table to cover the whole FK range.
  - The yellow row requires both house records to satistify it.
  - So how do you implement that?
    - You need to roll up all rows with the same scalar key part and combine their ranges into one big range.
      - Maybe the resulting range even has gaps.
      - Sounds like a job for ... multiranges!
      - And the way we roll up values from multiple rows is with aggregate functions.
      - So what we need is range_agg.



# Foreign Keys

```
CREATE CONSTRAINT TRIGGER RI_ConstraintTrigger_c_454735
  AFTER INSERT ON rooms FROM houses
  NOT DEFERRABLE INITIALLY IMMEDIATE
  FOR EACH ROW
  EXECUTE PROCEDURE RI_FKey_check_ins();
```

Note:

- Ordinary FKs are actually trigger, but they are special in two ways.
  - First, they are constraint triggers.
    - A constraint trigger is an ordinary trigger that can cancel a query, except like constraints it is deferrable.
  - Second, FKs are "internal" triggers, which means you don't see them when you describe a table, etc.
- They are AFTER ROW (like every constraint trigger).
- There are 2 on the referencing table and 2 on the referenced table.
  - Suppose we call these the "parent" and "child" tables.
  - If you delete or update a parent, we have to make sure you didn't create any orphans in the child table.
  - If you insert or update a child, we have to make sure its parent exists.
    - So I'm showing SQL equivalent to what Postgres does to create the insert trigger on the child table.
    - If you look in `pg_trigger` you can see it, and if you call `pg_get_triggerdef` on its oid you'll see this SQL.



# Foreign Keys

```
SELECT 1
FROM   houses x
WHERE  id = $1
FOR KEY SHARE OF x;
```

Note:

- Here is the SQL that trigger runs.
  - So we run this when the child table changes (the rooms table).
  - $1 is the FK value.
  - If we had multiple columns then you'd see a $2, $3, etc.
  - We're just making sure that the PK exists on the parent table.
- This is the "interesting" side of the relationship.
  - Checking the child table when a PK changes is easier.



# Foreign Keys

```
CREATE CONSTRAINT TRIGGER TRI_ConstraintTrigger_c_454735
  AFTER INSERT ON rooms FROM houses
  NOT DEFERRABLE INITIALLY IMMEDIATE
  FOR EACH ROW
  EXECUTE PROCEDURE TRI_FKey_check_ins();
```

Note:

- Temporal FKs are the same way but just slightly different trigger functions.
  - They start with TRI instead of RI (for Temporal Referential Integrity).
- Like before, the triggers just run some SQL to make sure the referential integrity still holds.



# Foreign Keys

```
SELECT 1
FROM (
    SELECT valid_at AS r
    FROM   houses x
    WHERE  id = $1
    AND    valid_at && $2
    FOR KEY SHARE OF x
) x1
HAVING $2 <@ range_agg(x1.r);
```

Note:

- Here is that SQL.
  - It's almost the same SQL as before.
- $1 is the new FK value.
  - We want to make sure the parent table's PK has that value too.
- First the pedantic stuff:
  - Obviously the table and column names are not hardcoded.
  - More than one scalar column is supported.
  - If it's not a partitioned table we say `FROM ONLY` (like other FK checks)
- $1 is the FK scalar value.
- After all the scalar keys, you have a $n+1, here $2.
  - That's the range value.
- Walk through:
  - So we pull out all the `valid_at` ranges for that ID...
  - As an optimization we only look at rows that overlap $2. Rows outside of the FK range are irrelevant.
  - We merge them together with range_agg...
  - and make sure they completely contain the FK range.
- This is trickier than it needs to be because of the locking.
  - `FOR KEY SHARE` is a very light lock really just used for FK checks.
  - It doesn't support aggregate queries though, so we have to aggregate outside that subquery.
- This query is for if the FK side changes.
  - Again, the query for if the PK side changes is simpler.
    - No need for `range_agg`.
    - It looks for FK rows that *still* match the old PK values.
      - If it finds any, the constraint fails.
    - In fact no changes needed at all.
      - = operator for the scalar parts
      - && operator for the range part



# Foreign Keys

<pre><code>CREATE CONSTRAINT fk_rooms_to_houses
  FOREIGN KEY
  (house_id, PERIOD valid_at)
  REFERENCES
  (id, PERIOD valid_at)
<div class="fragment">  ON DELETE CASCADE
  ON UPDATE CASCADE;</div></code></pre>

Note:

- So let's go back to our SQL statement....
- (flip to next slide)
- What about CASCADE?
  - What does it even mean?
    - I means we delete/update the affected region of the referencing table.
- Cascade is actually easy to implement *if* we have temporal update/delete.
  - A temporal update/delete is exactly what a temporal cascade does.
  - So let's go to that....



# UPDATE

```
UPDATE houses
  FOR PORTION OF valid_at
  FROM  '2020-01-01' TO '2021-01-01'
  SET   tax_appraisal = 250000
  WHERE id = 5;
```

![house update](img/house_update.png)

Note:

- Here is the syntax for a temporal update.
  - We add this FOR PORTION OF clause to say what timeframe we want to target.
- For example:
  - Suppose we have a house and we want to update the property tax appraisal for the year 2020.
  - We update just that timeframe.
- Here we see a before and after.
  - The green boxes are the same record, pre- and post-udpate.
  - The yellow boxes are newly inserted records.
    - The have the same values as the pre-update record, but their valid_at boundaries are changed.
  - So doing a temporal update can actually cause inserts to happen.
    - If you have insert triggers on the table, those fire on the secondary inserts!



# DELETE

```
DELETE FROM houses
  FOR PORTION OF valid_at
  FROM  '1950-01-01' TO '1960-01-01'
  WHERE id = 7;
```

![house delete](img/house_delete.png)

Note:

- Delete is very similar
  - Same syntax
- For example:
  - Suppose we discover a house wasn't built until 1960.
- According to the standard, we're supposed to really delete the record,
  - and then insert new records if there is anything left over.
  - So the yellow box is a newly inserted record.
  - If there were any leftovers *before* the deleted section, we'd insert a record for that too.
- So like update, delete can cause inserts to happen.
- In implementation UPDATE and DELETE are practically the same, so from now on I'll treat them together.



# UPDATE/DELETE

- Parse
- Analyze
- Rewrite
- Plan
- Optimize
- Execute

Note:

- unlike temporal integrity constraints, these go through the whole query pipeline.
  - making primary & foreign keys are "utility" commands so they skip the planner & optimizer.
  - But DML goes through all the steps.

- Parse: just pull out the strings into appropriate structs. Don't try to do anything yet.
- Analyze: make sure columns really exist, turn things into a Node tree for expressions, column references, function calls, operators.
- Rewrite: transform the query if there are views or rewrite rules.
- Plan: generate lots of possible plans.
  - Each plan is another tree of Nodes.
- Optimize: choose the plan that seems best.
  - If you `EXPLAIN` you can see the plan's node tree, basically.
- Execute: nodes that actually do everything.
  - In my case, everything else was easy, practically just copying things from one struct to another,
  - ...but here things got really hard.
    - TupleTableSlots
    - Locking and transaction isolation
  - I really owe my progress here to my wife, who let me go off to a cabin and work on it for a whole weekend, while she took care of our five kids.

- I've noticed on the mailing list that a lot of guest submissions get rejected because all the work happens in the wrong part of the process, e.g. in the analysis phase.
  - Someone wrote a great post there a year or two ago explaining why you couldn't do all the work in the analyze phase. I wish I had it bookmarked!



# UPDATE/DELETE: Parse

`parsenodes.h`

```
typedef struct ForPortionOfClause
{   
    NodeTag     type;
    char       *range_name;
    int         range_name_location;
    Node       *target_start;
    Node       *target_end;
} ForPortionOfClause;
```

Note:

- We fill this in in the bison file.



# UPDATE/DELETE: Parse
<!-- .slide: style="font-size: 95%" -->

```
for_portion_of_clause:
      FOR PORTION OF ColId FROM a_expr TO a_expr
        {
          ForPortionOfClause *n = makeNode(ForPortionOfClause);
          n->range_name = $4;
          n->range_name_location = @4;
          n->target_start = $6
          n->target_end = $8
          $$ = n;
        }
      | /*EMPTY*/         { $$ = NULL; }
    ;
```

Note:

- This is from `gram.y`
- `range_name` is a range column name (or in the future also a PERIOD name).
- `range_name_location` is an ordinary bison thing so that error messages can point out where the problem is.
- `target_start` and `target_end` are endpoints used to build another range.
  - Note that these take expressions.
  - At first I only accepted literal strings because a shift/reduce conflict made things tricky.
  - Obviously that's not sufficient.
    - You certainly need NOW() and time-related functions.
    - The spec disallows column references and non-deterministic functions,
      - so basically you're getting constant bounds.



# UPDATE/DELETE: Parse

```
UPDATE for_portion_of_test
FOR PORTION OF valid_at
  FROM '2018-03-01' AT TIME ZONE INTERVAL '1' DAY TO HOUR
    TO '2019-01-01'
    SET name = 'one^3'
    WHERE id = '[1,2)';
```

Note:

- Here is the problem
  - An interval can use this "to hour" syntax.
    - The "TO" has to be a smaller granularity than the interval itself.
      - So you can have 1 year to month, 1 day to hour, etc.
    - And you can use this to express the time zone offset.
  - I've never used this way of giving a time zone,
    but apparently it's from the SQL standard.
    - I'm not even sure what it means,
      or what purpose it serves.
      I couldn't find it in our documentation,
      but we have tests for it.
  - After bison reads the DAY in "DAY TO HOUR",
    should it keep parsing the interval (i.e. shift)
    or should it end the FROM part (reduce)?



# UPDATE/DELETE: Parse

```
state 1855

  1923 opt_interval: DAY_P .
  1928             | DAY_P . TO HOUR_P
  1929             | DAY_P . TO MINUTE_P
  1930             | DAY_P . TO interval_second

    TO  shift, and go to state 2611

    TO        [reduce using rule 1923 (opt_interval)]
    $default  reduce using rule 1923 (opt_interval)
```

Note:

- Here is bison's debugging output about the problem.
  - The dot is bison's current position.
    - So it's just read the token "DAY".
    - TO is the lookahead token.
    - Should it shift (i.e. keep going in the current rule)
      or reduce (i.e. finish the current rule about intervals)?
- If you *do* use this interval feature, there is no ambiguity, because we have TO twice.
  - We can see that, but bison can't because it has only one lookahead token.
- If you have TO once there is also no ambiguity: it must belong to FOR PORTION OF.
- But I don't know how to teach bison that.



# UPDATE/DELETE: Parse

```
if (foo)
  if (bar)
    stmt1
  else
    stmt2
```

Note:

- It's a lot like this classic shift/reduce problem:
  - Where does the else go?
- Historically we've always attached the else to the nearest if.
  - That's what I do as well:
    - The TO belongs to the interval.
      - I gave TO a higher precedence that the DAY/HOUR/etc tokens.
  - I think attaching the TO to the FOR PORTION OF would be better actually,
    since the interval feature is so obscure.
    - But I couldn't do that without breaking intervals everywhere.
  - I doubt it will be a common issue in practice.
    - Who writes time zones like this?
    - You should use time zone names instead of offsets anyway, so they work on both sides of Daylight Savings Time.
    - You can always add parens if you need to.
- I'll probably keep stubbornly trying to fix it,
  - But if anyone wants to help, let me know.
  - This isn't in a patch file yet, but it will be soon, and the code is in a branch on my personal github account.



# UPDATE/DELETE: Analyze

```
FOR PORTION OF valid_at
  FROM  '2020-05-30'
  TO    '2021-01-31'
```

Note:

- So we'll move on to the analyze phase.
- Here we copy stuff from the previous struct to a new one with more information.
  - We're doing things like looking up oids, finding the type of things, etc.
  - Lots of error-checking happens here.
    - Make sure valid_at is a column.
    - Make sure the column is a range column.
    - Make sure the table has a temporal PK.
    - Make sure the range is part of the PK.
- We build up some helpful expression nodes.



# UPDATE/DELETE: Analyze

```
FuncCall *fc = makeFuncCall(
        SystemFuncName(range_type_name),
        list_make2(result->targetStart,
                   result->targetEnd),
        forPortionOf->range_name_location);
result->targetRange = transformExpr(
        pstate,
        (Node *) fc,
        EXPR_KIND_UPDATE_PORTION);
result->overlapsExpr = (Node *) makeSimpleA_Expr(
        AEXPR_OP, "&&",
        (Node *) result->range,
        (Node *) fc,
        forPortionOf->range_name_location);
```

Note:

- So the analysis phase has to take the parse result and make real node trees.
- Here we have a function call node.
  - It will call the range constructor with the FOR PORTION OF endpoints.
  - We also build a Node to call the "overlaps" operator on that range and the range column.
    - The range column is a Var node and is stored in `result`'s `range` field (not shown here).



# UPDATE/DELETE: Analyze

```
if (stmt->forPortionOf)
{
    if (stmt->whereClause)
        whereClause = (Node *) makeBoolExpr(
            AND_EXPR,
            list_make2(qry->forPortionOf->overlapsExpr,
                       stmt->whereClause),
            -1);
    else
        whereClause = qry->forPortionOf->overlapsExpr;
}
else
    whereClause = stmt->whereClause;
```

Note:

- Here's another snippet from the analyze phase.
- We use FOR POTION OF to add an implicit condition to the WHERE clause.
  - We only want to hit records that overlap the range you're targeting.
  - This is close to the offense I described above; maybe it's even in the wrong place.
    - If you use `EXPLAIN` then you see this extra condition.
      - But in my opinion that's actually desirable here.



# UPDATE: Analyze
<!-- .slide: style="font-size: 85%" -->

```
Expr *rangeSetExpr = (Expr *) makeSimpleA_Expr(
        AEXPR_OP, "*",
        (Node *) result->range,
        (Node *) fc,
        forPortionOf->range_name_location);

rangeSetExpr = (Expr *) transformExpr(
        pstate,
        (Node *) rangeSetExpr,
        EXPR_KIND_UPDATE_PORTION);
TargetEntry *tle = makeTargetEntry(
        rangeSetExpr,
        range_attno,
        range_name,
        false);

targetList = lappend(targetList, tle);
```

Note:

- We also build a Node for the "intersects" operator,
  which we use to UPDATE the range column.
- fc is the range constructor call from above.
- The "target list" is all the things the UPDATE will update
  - (or that SELECT will select, incidentally).



# UPDATE/DELETE: Analyze

```
FOR PORTION OF valid_at
  FROM  '2020-05-30'
  TO    'Infinity'

FOR PORTION OF valid_at
  FROM  '2020-05-30'
  TO    NULL
```

Note:

- How should we deal with "update from now until further notice"?
  - This is really important: it's what you'll use more often than not.
- This isn't addressed specifically by the SQL standard.
- In the standard you're supposed to have a sentinel like January 1, 3000.
  - That's so ugly though.
  - 'Infinity' could make an okay sentinel.
  - It is a valid value for timestamps, dates, floats. Not ints.
    - It'd be nice to support all types, although officially we don't have to.
- With ranges you express "unbounded" with NULL.
- Oracle uses NULL the same way btw.
  - It lets you have PERIODs with NULL endpoints and treats them how our ranges treat NULLs: unbounded.
  - I imagine our PERIODs should be the same.
- But what do you write in the FOR PORTION OF?
  - Do you write NULL? That seems a little awkward.
    - Writing 'Infinity' might feel more natural.
  - Nevertheless I like NULL the best:
    - Fits better with ranges.
    - Works for all types.
    - But still I fear some people won't think of it.



# UPDATE/DELETE: Analyze

```
=# SELECT tstzrange(NOW(), NULL)
-#      - tstzrange(NOW(), 'infinity');
```

Note:

- Here's another issue.
- What do you suppose this will be?
  - We're taking a range that goes out to infinity (with NULL),
    and sutracting everything out to infinity (with the string Infinity).
  - Will it be the empty range?



# UPDATE/DELETE: Analyze

```
=# SELECT tstzrange(NOW(), NULL)
-#      - tstzrange(NOW(), 'infinity');
  ?column?   
-------------
 [infinity,)
(1 row)
```

Note:

- No! We're left with a tiny slice: the range from infinity and beyond.
- So using +/- Infinity causes problems because there is this tiny slice left over.
  - As your database ages, you're going to have a bunch of records with just this sliver of nonsense time.
- I used to have some code that detected literal `Infinity` and replaced it with NULL.
  - That kind of "helpfulness" seems contrary to the Postgres approach.
- But accepting Infinity here is surely a footgun.
  - But it's the same footgun that already exists with ranges.
    - I'm not really adding anything new.
  - I'm inclined to just leave it, but add a warning in the documentation.

  - This may be a bigger deal than it feels though (and it feels really small to me):
    - I gave a trial run of this talk last month at my local Postgres meetup,
      and someone said they had actually experienced the timestamp-vs-null problem with ranges,
      and it made a big mess of their data until they figured out that infinity and null aren't the same.
      So maybe we need to do something.

  - Maybe detect literal Infinity and print a warning?

- We could also have our own syntax...
  - where you can leave of `FROM` or `TO` entirely (or even both),
  - or where you have some keyword like INFINITY.
  - That's non-standard.
    - I don't plan to do it but I kind of like it.



# UPDATE/DELETE: Rewrite

Note:

- Nothing here!



# UPDATE/DELETE: Plan

Note:

- Nothing here!



# UPDATE/DELETE: Optimize

Note:

- Nothing here!



# UPDATE/DELETE: Execute
<!-- .slide: style="font-size: 85%" -->

```
// executor/modifyTableNode.c
ExprContext *econtext = GetPerTupleExprContext(estate);
econtext->ecxt_scantuple = slot;

ExprState *exprState = ExecPrepareExpr(
        (Expr *) forPortionOf->targetRange, estate);
targetRange = ExecEvalExpr(exprState, econtext, &isNull);

if (isNull) elog(ERROR, "Got a NULL FOR PORTION OF target range");
targetRangeType = DatumGetRangeTypeP(targetRange);
resultRelInfo->ri_forPortionOf->fp_targetRange = targetRangeType;
```

Note:

- In the execute phase, we have to do one extra thing:
  - Perform the extra inserts for leftovers before and after the targeting range.
- First we need to evaluate the target range.
  - Remember we built a Node for the range constructor function.
  - Now we actually evaluate the function.
  - We cache this so we don't have to do it every row.



# UPDATE/DELETE: Execute

```
table_tuple_fetch_row_version(
        resultRelInfo->ri_RelationDesc,
        tupleid, SnapshotAny, oldtupleSlot);
oldRange = slot_getattr(
        oldtupleSlot,
        forPortionOf->range_attno, &isNull);
oldRangeType = DatumGetRangeTypeP(oldRange);
```

Note:

- We need to get the `valid_at` range of the current tuple.
- Some error checking is omitted here....
- Honestly I barely understand the TupleTableSlots, but I think it's like this:
  - A tuple table is a collection of tuples, and a slot holds one tuple.
    - The slot isn't the tuple itself:
      - it has metadata about the tuple, like whether it's part of a table or just some expression, etc.
    - There are functions to store a tuple into a slot or get it out again.
  - We initialize a tuple table slot in `ExecInitModifyTable` which runs once for the whole statement,
    - And then we use it later once per row.
  - If someone knows of a good document I can read about tuple table slots, I'd love to know about it.
- So the first line stores the tuple into the slot based on tupleid.
  - I suspect `SnapshotAny` is wrong....
  - I was surprised the tuple we're currently updating wasn't already "loaded" and available somewhere, but I couldn't find it.
- Next we pull out a `Datum`, which is what `oldRange` is.
  - A `Datum` is just Postgres's generic type for any value.
  - There are macros to convert it to whatever type you like, here to a range type.



# UPDATE/DELETE: Execute

```
range_leftover_internal(
        typcache,
        oldRangeType,
        targetRangeType,
        &leftoverRangeType1,
        &leftoverRangeType2);
```

Note:

- I added this range function that does what we need.
- We find out if the current tuple's range extends past the FOR PORTION OF range,
  - either on the left or on the right.
  - Store the leftovers in `leftoverRangeType1` and `leftoverRangeType2`.
    - We need to insert those as new rows in the table.



# UPDATE/DELETE: Execute

```
MinimalTuple oldtuple = ExecFetchSlotMinimalTuple(
        oldtupleSlot, NULL);
ExecForceStoreMinimalTuple(oldtuple, leftoverTuple1, false);

leftoverTuple1->tts_values[forPortionOf->range_attno - 1] =
        RangeTypePGetDatum(leftoverRangeType1);
leftoverTuple1->tts_isnull[forPortionOf->range_attno - 1] =
        false;

ExecMaterializeSlot(leftoverTuple1);
ExecInsert(mtstate, leftoverTuple1, planSlot,
           estate, node->canSetTag);
```

Note:

- Suppose we have some leftovers before the target range.
  - We extract the current tuple from its slot.
  - We copy it into `leftoverTuple1` as a "minimal tuple" which means we can manipulate its Datums in memory.
  - Then we set the Datum for the `valid_at` column.
  - Then we insert the tuple into the table.
- Similar code handles leftovers after the target range.

- To be honest, I wasn't totally happy with all this.
  - It felt too specific for being in this modify table node, like I was mixing widely different levels of abstraction.
    - Nothing else in there references specific types, for instance.
    - I had to add several `#include` statements to the top to give myself access to the functions I needed.

- It was fun to learn about executor nodes, but maybe it's the wrong approach.



# UPDATE/DELETE: Execute

<pre><code>typedef struct TriggerData
{
    NodeTag          type;
    TriggerEvent     tg_event;
    Relation         tg_relation;
    HeapTuple        tg_trigtuple;
    HeapTuple        tg_newtuple;
    Trigger         *tg_trigger;
    TupleTableSlot  *tg_trigslot;
    TupleTableSlot  *tg_newslot;
    Tuplestorestate *tg_oldtable;
    Tuplestorestate *tg_newtable;
<div class="fragment">    RangeTypeP      *tg_targetportion;
</div>} TriggerData;</code></pre>

Note:

- And actually I think I could rip out all these executor changes,
  if only I could implement the secondary INSERTs in a trigger.
  - We saw how FKs use hidden triggers already.
  - This would also be an AFTER ROW trigger.
  - I just need some way to tell the trigger what was in the FOR PORTION OF clause,
    - and whether that clause was even used.
      - It needs to know whether this a temporal UPDATE/DELETE or a normal one.
      - Right now there's no way to do that.

- This struct is info that gets passed to every trigger function.
  - In C you get this struct directly.
  - In plpgsql these are available as `TG_*` variables.
- So what if we added the FOR PORTION OF target to the struct?
  - Maybe it should be a Datum instead?
    - That feels more correct to me.
    - If so then I need a second field saying whether it's NULL (i.e. FOR PORTION OF wasn't used).

- My current patch has the executor node approach, but I plan to try out this trigger approach instead.
  - Let me know what you think!



# Thanks!

https://github.com/pjungwir/pg-temporal-talk-2020

Note:

- These slides are available on Github.
- If you have any feedback or questions I'd be glad to hear it.
