## Boogie

Boogie consists of two parts, mathematical and imperative.

### Math
```
type
const
function
axiom
```
Axioms are used to reason about the type declarations, constants and first order functions.

### Imperative

#### Expressions
All expressions are total. Even division by zero results in some fixed value based on its arguments.

Maps.


A procedure declaration:
```
procedure P(ins) returns (outs); Spec
```
- precondition
- frame
- postcondition
- old

A procedure implementation contains the following:
- any number of in/out params, with in-parameters being immutable
- declaration of a number of local variables
- list of statements


Grammar of statements:

```
Stmt ::= xs := Exprs;
	| x[Exprs] := Expr;
	| havoc xs;
	| if (Expr) { Stmts } else { Stmts }
	| while (Expr) Invs { Stmts }
	| assert Expr;
	| assume Expr;
	| call xs := P(Exprs);
```
x: identifier <br/>
xs: list of identifiers <br/>
P: name of a declared procedure <br/>
Expr: an expression <br/>
Stmts: a list of statements <br/>
Invs: loop invariant declarations, for example: `invariant Expr;` <br/>

Loop invariants must hold at the point immediately before each evaluation of the loop guard. <br />
Otherwise, execution of the loop results in an irrecoverable error.



*havoc* takes a list of variables and assigns each one of them to an arbitrary value
```
havoc x;
```

*assert* represents a condition that must hold at the program point for a program to be correct. <br />
If it holds, the statement acts like a no-op. <br />
Otherwise, it results an irrecoverable error.
 

*assume* expresses that the verifier should only consider executions where the given condition holds.
If it holds, the statement acts like a no-op.

The following example sets *x* to an arbitrary value but executions are only considered by the verifier for values of x that are greater than 0.
```
havoc x; assume x > 0;
```

### Semantics
It is inductively defined in terms of weakest preconditions, on the structure of basic statements. <br />
`wp[S, Q]`: weakest precondition of S w.r.t Q, where S is a statement and Q is a condition on the post-state of S. Q is satisfied if the execution of S terminates. Weakest preconditions express what must hold in the pre-state of S in order for Q to be satisfied.

#### Simple Statements
- `wp[xs := EE;, Q] = Q[EE/xs]`  (Q is established if what Q says about xs holds for EE in pre-state)
- `wp[havoc xs;, Q] = forall xs, Q.` (values of xs are chosen to satisfy Q)
- `wp[assert E;, Q] = E && Q`
- `wp[assume E;, Q] = E => Q`
- `wp[S T, Q] = wp[S, wp[T, Q]]`
- `wp[if (E) { S } else { T }, Q] = (E => wp[S, Q]) && !(E => wp[T, Q])`
- `wp[m := m[jj := E];, Q] = Q[m[jj := E] / m]` (map update)

#### Loops
A loop keeps iterating as long as its guard is satisfied. As such, the iterations of a loop are abstracted and its semantics defined in terms of invariants. This is essentially verifying a supposed fix-point (?).

Sources of invariants:
- properties that always hold in source language (eg. an allocated object remains allocated which is a property of Dafny)
- rules enforced by programming discipline (eg. an object can only change if it is included in modifies clause or if newly allocated)
- invariants inferred by an inference engine (see *abstract interpretation*)
- invariant declared by programmer

Let the *syntactic assignment targets* of a statement S denote the set of variables to which S assigns.

The meaning of:
```
while (E) invariant J; { S } 
```
Is encoded as:
```
assert J; // check loop invariant holds on entry
// xs denotes the syntactic assignment targets of S. They are assigned arbitrary values such that J holds.
// the loop is fast forwarded to top of arbitrary loop iteration
havoc xs; assume J; 
if (E) { // if E does not hold, the loop terminates and execution continues.
	S assert J; // check that invariant is maintained after executing loop body
	assume false; // assume false has the result of ignoring executions that go correctly as only executions that are incorrect matter
} else {}
```

#### Procedure Calls
Procedure calls are reasoned about in terms of their specification, not implementation because:
- it avoids issues of fix-points (?)
- allows data abstraction

The following is a procedure declaration:
```
procedure P(ins) returns (outs);
		requires pre;
		modifies gs;
		ensures Post;
```

That is called as follows:
```
call xs := P(EE);
```

With the call’s meaning encoded as:
```
ins' := EE; // evaluate in-parameters
assert Pre'; // check precondition
gs' := gs; // remember values of old variables in modifies clause
havoc gs, outs'; assume Post’; // set out-params and modified global variables to arbitrary values such that post condition holds
xs := outs' // set actual out-params from formal out-params
```

The weakest preconditions are computed from these statements.


#### Procedure Implementations
A verification condition for a procedure is generated based on:
- postconditions of the procedure being verified
- preconditions of procedures that are called
- conditions in assert statements

Only *modifies* clauses are checked syntactically. <br />
All global variables which are assigned in the body must be listed in the *modifies* clause. 
 
Preconditions are assumed to be held at the start of the implementation body. <br />
They are akin to *assume* statements.

Postconditions are checked at the end of the procedure. <br />
They are akin to *assert* statements.

Consider the following procedure declaration:
```
procedure P(ins) returns (outs);
		requires pre;
		modifies gs;
		ensures Post;
```

And the following implementation:
```
var locals; stmts
```

Whose meaning is encoded as the following, denoted by *Impl*:
``` 
assume Pre;
gs’; // a list of fresh variables, one for each in gs
stmts’; // each statement in stmts is expanded with its semantic encoding
assert Post’;
```
For stmts’ and Post’, each occurrence of a variable from gs inside an old expression is replaced by the corresponding variable from gs. Then, every *old(E)* is replaced by E. This obtains the pre-state value of a variable if it is in the modifies clause. Otherwise, it obtains the current value of the variable. 


Let *Axs* denote the conjunction of axioms in the program. The verification condition for the procedure is represented by:
```
Axs => wp[Impl, true] // under the given axioms, the implementation executes correctly
```

## Translating Dafny to Boogie

Grammar of a Dafny program:
```
Program ::= Classes
Class ::= class Id { Members }
Member ::= Field | Method | Function
```
A set of named classes make up a Dafny program.


### Declarations
A translation begins with the following declarations:
- ones that encode properties that hold for all Dafny programs
- a single declaration for each class

#### Classes
A type is declared in the translation, `type <ClassName>`


A class as a whole only contributes a single constant representing its name to the Boogie translation. <br />
The following denotes the translation of a class declaration:
```
decl[class C { <members> }] = const unique class.C: <ClassName>; decl*[ <members> ]
```
`class.C` is the name of the constant (Boogie allows non-alphanumeric characters for identifiers). <br />
`decl*[ <members> ]` denotes the application of `decl` to every member of the class.

#### Types
`type Ref;` is a nullary type constructor for references <br />
`const null: Ref;` represents Dafny’s `null` reference <br />
`type Set a;` is a unary type constructor for sets <br />
`type Seq a;` is a unary type constructor for sequences <br />

| Dafny         | Boogie        |
|:-------------:|:-------------:|
| bool         | bool        |
| int      | int     |
| *Id* | *Ref* |
| object | *Ref*
| set[T] | *Set* type[T] |
| seq[T] | *Seq* type[T] |

`type[T]` maps the Dafny type `T` into its corresponding Boogie type.

Since all Dafny class types are represented by the Boogie type *Ref*, references of different *Dafny* types need to be distinguished. The translation includes a function to map each reference to its allocated type:
```
function dtype(Ref) returns (<ClassName>)
```

The translation also needs to include operations on sets (based on set membership) and sequences.

### Memory
Dafny includes dynamically allocated objects and references to these objects.

Memory is modelled as a polymorphic map. <br />
Its keys are pairs of object references and field names. <br />
Its values are the values of the corresponding object and field.

The following declaration are introduced during the translation to Boogie:
- `type Field a`
- `type HeapType = <a>[Ref, Field a]a;`
- `var H: HeapType`

*H* is the global variable which is a map.

Any field *f* in a class *C* is translated as follows:
```
decl [var f: T;] = const unique C.f: Field type[T]
```
Each field in a Dafny program corresponds to a unique value of the appropriate *Field* type.

The map can contain allocated as well as unallocated references. <br />
To distinguish between these, a ghost boolean field is added:
```
const unique alloc: Field bool;
```

To distnguish the map representing the heap and other maps, a predicate is introduced by the translation:
```
function GoodHeap(HeapType) returns (bool);
```

There are various axioms that state properties which hold for all heaps. The following is one example.

For a class *C* with a field *f* of reference type *D*, *C.f*:
- yields a correctly typed value
- is closed under allocation (an allocated object only reaches other allocated objects)
```
axiom (
	forall h: HeapType, o: Ref,
	GoodHeap(h) && o != null && h[o, alloc] => GoodRef[h[o, C.f], D, h]
);

GoodRef[t, T, h] =
	t == null \/ (h[t, alloc] /\ dtype(t) == class.T), if T is a class name
	t == null \/ h[t, alloc], if T is "object”
```


### Expressions 

|   Dafny        | df        | tr | explanation |
|:-------------: |:-------------:|:-------------:| :-------------:|
|   x    | true        | x        | |
| this      | true     | this     | |
| E + F | df[E] /\ df[F] |  tr[E] + tr[F] | |
|  E / F | df[E] /\ df[F] /\ tr[F] != 0 | tr[E] / tr[F] | F should not evaluate to 0|
| E /\ F | df[E] /\ (tr[E] => df[F]) | tr[E] /\ tr[F | Short circuiting is employed. The definedness of F matters only if E evaluates to true |
| E.f | df[E] /\ tr[E] != null | *H*[tr[E], C.f] | *E.f* refers to member selection in Dafny. *C.f* refers to the name of a field, in Boogie. The object whose field is being selected must not be null |

*df* is a function taking in an expression and returning a predicate indicating whether or not the expression is well defined in Dafny. <br />
*tr* is a function taking in a well defined expression that returns the value of the expression. <br />

What does it mean for an expression to be well defined?

Expressions which refer to both the current state as well as the initial state of the method:

|   Dafny        | df        | tr | 
|:-------------: |:-------------:|:-------------:| 
| old(E)      | old(df(E))	| old(tr[E])     |
| fresh(E) | df[E] |  forall o: Ref, o ε tr[e] => o == null \/ !old(H)[o.alloc] |



### Methods
A method in Dafny is declared within a class. It consists of a specification and an implementation:
```
Method ::= method Id(Params) returns (Params) Specs { Stmts }
Param ::= Id : Type
Spec ::= requires Expr ; | modifies Exprs; | ensures Expr ;
```

A method is translated into a Boogie procedure. The implicit receiver parameter *this* is made explicit as a result. 
It can also allocate new objects and modify them. Therefore the heap is implicitly allowed to be modified in any procedure. 
*modifies* takes a list of reference type expressions as opposed to a single one in Boogie.<br />

The following contributes to the specification of a Boogie procedure:
- the Dafny method’s specification
- parameter types
- properties that hold for all Dafny programs
	- for example, that *this* is allocated: `this != null && H[this, alloc]` 
	- for example, that allocated objects remain allocated
	- these are proved only once to save time
	- after being proved, they are introduced in the translation as *free* conditions which means they are assumed and not checked
- details of the encoding of Dafny expressions/statements


| Dafny         | Boogie        |
|:-------------:|:-------------:|
| requires *Pre*         |   free requires df[*Pre*] <br /> requires tr[*Pre*]     |
| ensures *Post*      | free ensures df[*Post*] <br /> ensures tr[*Post*]     |

The definedness of the pre/post conditions is marked as *free* as it is checked in a separate procedure. This is to avoid having to verify the definedness of Pre at every call site. 


Example:
```
decl[ method M(ins) returns (outs) requires Pre; modifies mts; ensures Post; { stmts } ]
=
procedure C .M (this: Ref , decl∗[ ins ]) returns (decl∗[ outs ])
free requires GoodHeap(H) && CanAssumeFunctionDefs; // use function axioms to reason about method bodies
free requires this != null && GoodRef[this,C,H]; 
free requires isAllocated∗[ ins ]; // ensure parameters are allocated
free requires df[ Pre ];
requires tr[ Pre ];
modifies H;
free ensures GoodHeap(H); // heap properties are ensured upon exit
free ensures boilerplatemts [ old(H) ]; 
free ensures isAllocated∗[ outs ]; 
free ensures df[ Post ];
ensures tr[ Post ];
{
	varlocals∗[stmts]; 
	stmt∗mts[stmts]
} 
```

```
decl[x: T] = x: type[T]
```
```
isAllocated[x:T] =􏰁 GoodRef[x,T,H] if T is a reference type true otherwise
```
Let *prevHeap* be *old(H)*.
```
boilerplate[prevHeap] =
	( ∀ ⟨a⟩ o: Ref , f : Field a • H[o, f] = prevHeap[o, f] || CanWrite[o] ) && // object reference o can only change if o is included in  modifies clause or if o is a newly allocated object
	(∀ o:Ref • prevHeap[o,alloc] => H[o,alloc]) // allocated objects should remain allocated
```
Let *mts* be the set denoted by the modifies clause, interpreted in the pre-state.
```
CanWrite[o] =
o ∈ old(tr[mts]) || ¬old(H)[o, alloc]
```


#### Functions
```
Function ::= function Id(Params): Type FSpecs { Expr }
Param ::= Id: Type
FSpec ::= requires Expr ; | reads Exprs ;
```

Example:
```
function F(ins): T requires R; reads rd; { body } 
```

Translation of a Dafny function results in the following:
- a Boogie function
- a Boogie procedure
- an axiom gives a precise definition of the value returned by the function by making use of the precondition and body
- a frame axiom which helps to prove (especially for recursive functions) that heap changes do not affect the function value by specifying the portion of memory the function depends on


A function declared in some class C:
```
function F(ins): T requires R; reads rd; { body }
```
Is translated as follows:
```
decl[function F …] = function C.F(h: HeapType, this: Ref, decl*[ins]) returns (type[T]);
```
```
df[E.F(EE)] = df[E] /\ df*[EE] /\ tr[E] != null /\ df[R[EE/ins]] /\ tr[R[EE/ins]]

tr[E.F(EE)] = C.F(H, tr[E], tr*[EE])
```
what do E and R refer to ?


Along with the following axiom:
```
// this axiom gives a precise definition of value returned by function using the precondition and body
// other axioms depend on it to ensure consistency of the definition (how?)
// this axiom is used by methods as a precondition  

axiom CanAssumeFunctionDefs =>
	(forall H: HeapType, this: Ref, decl*[ins]
		GoodHeap(H) /\ this != null /\ df[R] /\ tr[R] => C.F(H, this, ins) = tr[Body]
	)
```
And the following Boogie procedure which corresponds to a proof obligation that all calls go to functions with a strictly smaller *reads* clause
```
procedure C.F WellDefined(this: Ref, decl*[ins])
	free requires GoodHeap(H)
	free requires this != null /\ GoodRef[this, C.H]
	free requires isAllocated*[ins]
{
	assume df[R] /\ tr[R];
	assert funcdf[body]; // funcdf is like df but for field selection and function calls which check that heap is read according to a given reads clause
}
```

