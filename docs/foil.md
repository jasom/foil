## Foil - a Foreign Object Interface for Lisp

##### Copyright (c) Rich Hickey and Eric Thorsen. All rights reserved.

The use and distribution terms for this software are covered by the [Common 
Public License 1.0](http://opensource.org/licenses/cpl.php), which can be found in the file CPL.TXT at the root of 
this distribution. By using this software in any fashion, you are agreeing to be 
bound by the terms of this license. You must not remove this notice, or any 
other, from this software.

## Contents

*   [Description](#description)
*   [Download](#download)
*   [Quick Start](#quickstart)
*   [API Reference](#api)
    *   [Foreign VMs](#fvms)
    *   [Foreign References](#frefs)
    *   [Wrapper Generation](#wrappergen)
    *   [Object Creation](#objects)
    *   [Object Services](#objectservices)
    *   [Vectors](#vectors)
    *   [Miscellaneous](#misc)
    *   [Proxies and Callbacks](#proxies)
    *   [Marshalling](#marshalling)
*   [Runtime Servers](#runtimeservers)
*   [Protocol](#protocol)
*   [Summary](#summary)

<a name="description"></a>

## Description and Rationale

Foil consists of a protocol and a set of libraries that facilitate 
access to popular object runtimes, such as the JVM and the CLI/CLR, and 
their libraries, from Lisp.  A protocol is defined which abstracts out the 
common features provided by Java-like environments - object construction, 
method, field, and property access, object lifetime management etc.  The 
protocol defines a set of features as well as an s-expression based stream 
format for communication.  Runtime server applications are provided that 
utilize Java and C# libraries to implement the object runtime side of the 
protocol for Java and the CLI.  Source for the applications is provided so 
that custom hosts can be built.  A library for Common Lisp is provided 
that implements the consumer side of the protocol, and offers seamless 
access to the foreign objects in a Lisp-like manner.  

The design of Foil owes much to [jfli](http://jfli.sourceforge.net/), an in-process solution to the same 
problem for Java, and it remains extremely similar in its Lisp interface.  
Several factors motivated the significant difference in the Foil design -
its use of an out-of-process instance of the foreign runtime: 

*   jfli did not see wide porting, due to its use of LispWorks' sophisticated FLI to access JNI,    and the lack of corresponding facilities in some other Lisps
*   I found that I needed to access already-running instances of the JVM,    for instance servlet containers, as done in [Lisplets](http://lisplets.sourceforge.net/), and felt
    I could accomplish similar things with less effort with Foil + marshallers
*   I wanted to access the CLR/CLI in a similar fashion to Java
*   It allows for more flexibility in dealing with threading issues

The major tradeoff in stream-based access to out-of-proc runtimes is a 
significant drop in per-call performance.  However, even with jfli, which 
was very fast, the overhead of reflection per call could be high in 
certain scenarios, since the APIs of these platforms tend to be very 
'chatty'.  Foil includes a marshalling system that allows for efficient 
transfer of large and composite objects with minimal call overhead, in a 
manner that doesn't pollute the Lisp code on the consumer side.  

#### Foil provides all the facilities of jfli and more - 

##### Features of jfli that are retained/enhanced:

*   Automatic function generation for constructors, fields, methods, and properties either by			named class, or entire package (sub)trees given a jar file or assembly name.
*   Java/CLI -> Lisp package and name mapping with an eye towards lack of surprise, lack			of conflict, and useful editor completion.
*   setf-able setter generation for fields and properties*   Java/CLI vector creation and aref-like access to Java/CLI vectors.
*   Constructors that allow for keyword-style property initialization.
*   Typed references to Java/CLI objects with an inheritance hierarchy on the Lisp side			mirroring that on the Java/CLI side - allowing for Lisp methods specialized on Java/CLI			class and interface types.
*   Implementation of arbitrary Java/CLI interfaces in Lisp, and callbacks from Java/CLI to			Lisp via those interfaces.*   Automatic lifetime maintenance of Lisp-referenced Java/CLI objects, boxing/unboxing			of primitive args/returns, string conversions, Java/CLI exception handling, overload			resolution etc.

##### Some of the additions are:

*   (Hopefully) Much improved portability (n.b. it has not been ported, but is mostly standard CL)
*   Access to the CLR with the same API
*   Support for CLR and JavaBean properties
*   Simultaneous access to multiple runtimes
*   Simultaneous access to the CLR and Java
*   A marshalling system which can, in a single call, pull across the types, hashcodes, and/or values ofreference objects to an arbitrary depth, with user customizable value marshallers.
*   All references to the same remote object are `eq` on the Lisp side
*   `ensure-typed-ref`, which makes a remote referenceits most fully derived type in Lisp, works in place, using `change-class`
*   vector argument boxing, so lightweight vectors-as-arguments can be created in-place		without the overhead of multiple calls to create and initialize the vector
*   keyword-style init of properties in constructor calls is supported by the ctor functions, and can be leveraged		in apply and mapping scenarios (this feature was limited in jfli to the `new` macro)

	<a name="download"></a> 

### Download and Communication

[Foil is hosted on SourceForge](http://sourceforge.net/projects/foil/) 
[![SourceForge.net Logo](http://sourceforge.net/sflogo.php?group_id=125543&amp;type=1)](http://sourceforge.net)

We are going to try using SourceForge facilities for all communication regarding Foil, 
so please use the project tracker and forums there.

<a name="quickstart"></a> 

### Quick Start

Build and start the Java or CLI [runtime server](#runtimeservers) of your choice.

    (compile-file "/dev/foil/foil.lisp")
    (load "/dev/foil/foil")
    (use-package :foil)
    ;this is specific to LispWorks' sockets support
    (require "comm")
    ;create a foreign VM
    (setf *fvm* (make-instance 'foreign-vm
                               :stream
                               (comm:open-tcp-stream "localhost" 13579)))
    
    ;;;;;;;;; if Java ;;;;;;;;;;;
    ;create a wrapper for dialog class
    (def-foil-class "javax.swing.JOptionPane")
    ;use the wrapper
    (use-package "javax.swing")
    ;show it
    (joptionpane.showmessagedialog nil "Hello World from Lisp")
    
    ;;;;;;;;; if CLI ;;;;;;;;;;;;
    ;create a wrapper for dialog class
    (def-foil-class "System.Windows.Forms.MessageBox")
    ;use the wrapper
    (use-package "System.Windows.Forms")
    ;show it
    (messagebox.show "Hello World from Lisp")

Typically you wouldn't define single class wrappers by hand, but would instead pre-generate wrappers for entire packages using
`get-library-classnames` and `dump-wrapper-defs-to-file`. See `foil-java.lisp` and 
`foil-cli.lisp` for examples.

<a name="api"></a> 

## Lisp API Reference

   Other than 2 wrapper-generator helpers, all of the code for the Lisp side of foil resides in `foil.lisp`, and is 
runtime server independent. Foil is designed to be very portable, and is 99% pure Common Lisp. A complete port requires some 
facility for weak hash tables and object finalization.

<a name="fvms"></a> 

### Foreign VMs

Foil is built upon the notion of interactions with one or more foreign VMs, 
instances of the JVM or CLR, running the Foil libraries, in another process on 
the same or another machine. The connection to a specific VM is via one or more 
bidirectional streams. Note that the instantiation of the foreign VM and the 
establishment of the streams is outside the scope of this Lisp API. It is 
presumed you might utilize one of the supplied runtime servers, creating stream 
connections via sockets or pipes with the API provided by your Lisp 
implementation. Many scenarios are possible, including embedding the Foil 
support libraries into your existing Java or C# application, multiple streams to different threads in the same VM, etc.

A foreign VM is represented by an instance of the `foreign-vm` class. Each instance has a primary default stream 
over which communication will occur. The special variable `*fvm*` represents the default VM to which any unqualified
Foil calls will be directed, and can be bound in a specific context, thus allowing for multiple VMs. 
Note - instance property/method calls will always be routed to the VM hosting that instance.

Foreign VMs maintain their own foreign reference pools, type caches etc, and objects from one VM cannot be passed to another, 
even if they are both Java or CLI. However, in multi-thread, multi-stream scenarios, references are valid across threads in the same VM,
and the runtime server implementations are thread safe.

A simple startup scenario would look like this:

* First, outside of Lisp, start the Java or CLI Foil runtime server supplied with Foil, running on port 13579, then:

   <pre>
   (load "/dev/foil/foil")
   (use-package :foil)
   (require "comm") ;LispWorks-specific socket library
   (setf *fvm* (make-instance 'foreign-vm
                           :stream
                           (comm:open-tcp-stream "localhost" 13579)))
    ;use Foil  </pre>

*   **Special Variable** `*fvm*`

    This must be bound to an instance of `foreign-vm`. Default: nil
 Direct use of this other than during initial setup is not recommended, use instead `with-vm` or `with-vm-of`

*   **Special Variable** `*thread-fvm*`

	If set, this thread is waiting on a callback from this VM. Default: nil
    This is only used for advanced multi-thread scenarios

*   **Special Variable** `*thread-fvm-stream*`

		    If this thread is waiting on a callback (i.e. `*thread-fvm*` is bound), and `(eql *fvm* *thread-fvm*)`,            use this stream instead of the primary default stream for the VM. Default: nil
This is only used for advanced multi-thread scenarios

*   **Class** `foreign-vm`

    Manages a foreign VM. Requires the initarg `:stream` be set to a bidirectional stream with an instance of the            Foil runtime services on the other end.

*   **Macro** `(with-vm vm &body body)`

    Causes the body to be evaluated in a context where `*fvm*` is bound to `vm`

*   **Macro** `(with-vm-of this &body body)`

    Causes the body to be evaluated in a context where `*fvm*` is bound to the source VM of `this`

<a name="frefs"></a> 

### Foreign References

Foil programs invariably create instances of objects in the foreign VM. Those objects are tracked by Lisp in instances 
of the `fref` class. The Foil API and the runtime server cooperate to ensure that objects referenced by Lisp are kept alive
in the foreign runtime, and that when no longer referenced by Lisp, they become available for collection in the foreign VM. 
`frefs` maintain their source VM, an ID and revision counter for this purpose. In addition, `frefs` can 
cache hash codes, types, and values that have been marshalled. Only a single `fref` will be created for each remote object,
thus any 2 `frefs` that reference the same object are themselves `eq`.

*   **Class** `fref`

    Reference to a foreign object. `fref` is the superclass of all of the Foil classes generated to mirror the
    foreign hierarchy.

*   **Method** `(fref-vm fref)` ->The foreign-vm from which this reference originated
*   **Method** `(fref-id fref)` ->An integer ID, unique within a VM
*   **Method** `(fref-type fref)` ->A Class or Type fref

    This will only be set if the type has been marshalled or `get-type` has been called on this fref.

*   **Method** `(fref-hash fref)` ->int

    This will only be set if the hash code has been marshalled or `hash` has been called on this fref.

*   **Method** `(fref-val fref)` -> A Lisp object representing the value of the object

    This will only be set if the value has been marshalled.

*   **Function** `(ensure-typed-reference fref)` -> `fref`, whose class may have been changed

    Given a generic Foil fref, determines the full type of the object and			uses `change-class` to convert the fref to that type. Since we don't want to always incur the cost of                        type determination, the wrapper-generated API functions return generic references. Use this function to convert to			a typed reference corresponding to the full actual type of the object when desired:
        CL-USER 42 > (setf string-class (get-type-for-name "java.lang.String"))
        #}1

        CL-USER 43 > (type-of string-class)
        FREF

        CL-USER 44 > (ensure-typed-ref string-class)
        #}1

        CL-USER 45 > (type-of string-class)
        CLASS.

<a name="wrappergen"></a> 

### Wrapper Generation

*   **Macro** `(def-foil-class full-class-name) -> unspecified`

    Given the package-qualified, case-correct name of a Java/CLI class as a string, will			generate wrapper functions for its public constructors, fields, properties and methods.

    The core API for generating interfaces to Java/CLI is the `def-foil-class` macro. This			macro will, at expansion time, use Java/CLI reflection to find all of the public			constructors, fields, properties and methods of the given class and generate functions to			access them.

#### The Generated API
When you e.g. `(def-foil-class "java.lang.ClassName")`			you get several symbols/functions:

*   A package named `|java.lang|` (note case)

    from which the following are exported:
    *  A class-symbol: `classname.` (note the dot is part of the name)
    which can usually be used where a typename is required. It also serves as the					name of the Lisp typed reference class.
    *   Every non-interface class with a public constructor will get:
        *   A constructor, `(classname.new &rest args) -> fref`, which							returns a foreign-reference (fref) to the newly created object. Note that the constructor function,
            and therefore everything built upon it, can take the actual arguments to the Java/CLI ctor, followed by
            zero or more property initializers, which take the form:
        
            :keywordized-propertyname value

            e.g. `(window.new parent :width 200 :height 200)`
                    thus supporting the creation and some setup of a new object in a single call
        *   A method defined on [`make-new`](#makenew), ultimately							calling `classname.new`, specialized on (the value of) the class-symbol
        
            Note that if the constructor is overloaded, there is just one function generated,					which handles overload resolution. The function documentation string describes					the constructor signature(s) from the Java/CLI perspective. The same argument					conversions are performed as are for fields (see below).
*   All public fields will get a getter function:

    `(classname.fieldname [instance]) -> field value`

    and a setter:

    `(setf classname.fieldname [instance])`

    Instance field wrappers take a first arg which is the instance. Static fields					get a symbol-macro `*classname.fieldname*`

    If the type of the field is primitive, the field value will be converted to a					native Lisp value. If it is a Java/CLI String, it will be converted to a Lisp string.					Otherwise, a foreign reference to the Java/CLI object is returned. Similarly, when					setting, Lisp values will be accepted for primitives, Lisp strings for Strings,					or foreign references for reference types.

*   All public properties (explicit properties in the case of the CLI,					implied properties in the case of Java as specified by the JavaBeans protocol)					will get a getter function if the property supports get:

    `(classname.propertyname [instance] [args]) -> property value`

    and a setter if the property supports set:

    `(setf classname.propertyname [instance] [args])`

    Instance property wrappers take a first arg which is the instance. Static properties					get a symbol-macro `*classname.propertyname*`
*   Every public method will get a wrapper function:

    `(classname.methodname &rest args) -> return-value`

    As with constructors, if a method is overloaded a single wrapper is created that					handles overload resolution.
    The same argument and return value conversions are performed as are for fields.					The function documentation string describes the method signature(s) from the					Java/CLI perspective.

*   A Lisp class with the class-symbol as its name. It will have as its superclasses					other Lisp classes corresponding to the Java/CLI superclass/superinterfaces, some of					which may be forward-referenced-classes, and will be ultimately derived from `fref`.                    An instance of this class will be					returned by `ensure-typed-ref`, at which point the entire hierarchy will					consist of finalized standard-classes.
*   Note that, due to the need to reference other Java/CLI types during the definition					of a class wrapper, symbols, classes, and packages relating to those other types					may also be created. In all cases they will be created with names and					packages as described above.

*   **Function** `(get-library-classnames jar-or-assembly-filename &rest packages)			-> list-of-strings`

    Returns a list of class name strings. Packages should be strings of the form "java/lang" or "System/IO"
for recursive lookup and "java/util/" or "System/IO/" (note trailing slash) for non-recursive.
*   **Function** `(dump-wrapper-defs-to-file filename classnames)  ->			filename`

    Given a list of classnames (say from `get-library-classnames`), writes			the consolidated expansions of calls to `def-foil-class` to a file:

    (dump-wrapper-defs-to-file "/lisp/java-lang.lisp"
      (get-library-classnames "/j2sdk1.4.2_01/jre/lib/rt.jar " "java/lang/"))
    (compile-file "/lisp/java-lang")
    (load "/lisp/java-lang")
    (use-package "java.lang")
    ;Wrappers for all of java.lang are now available
This is the recommended way to access entire library packages. In particular, it has the advantage that the dumped
code does not require a foreign runtime to either compile or load.

    <a name="objects"></a> 

### Object Creation

In addition to the generated ctor wrappers (`classname.new` described above), the following, built upon the same,
add some additional capabilites and ease of use:

*   **<a name="makenew"></a>Generic Function** `(make-new classname. &rest args) -> fref`

    Allows for definition of before/after methods on constructors. Calls
    `classname.new` ctor.  The new macro expands into a call to this.

*   **Macro** `(new class-spec args &body body) -> fref`

    `class-spec -> class-sym | (class-sym this-name)`

    `class-sym -> classname.`

    `args -> as per ctors and make-new`

    Creates a new instance of class, using the `make-new` generic function,
then runs the body replacing all top-level calls of the form
`(.anything whatever)`
with
`(classname.anything new-object whatever)`
If `this-name` is supplied it will be bound to the newly-allocated object and available
to the body (note - but not to the args!)
Example:
<pre>
(new shell. (*display* :text "SWT Apropos" :layout (gridlayout.new 1 t ))
              (.setsize 800 600)
              (.setlocation 100 100))            </pre>
Expands into:
<pre>
(LET ((#:G2249 (MAKE-NEW SHELL. *DISPLAY* :TEXT "SWT Apropos" :LAYOUT (GRIDLAYOUT.NEW 1 T))))
  (PROGN
    (SHELL.SETSIZE #:G2249 800 600)
    (SHELL.SETLOCATION #:G2249 100 100))
  #:G2249)</pre>

    <a name="objectservices"></a> 

### Object Services

These functions provide 
access to basic facilities provided by all runtimes (usually through syntax or 
the Object class), but should be used instead, as they are portable and can be 
more efficient, caching and resolving some things locally on the Lisp side.

*   **Function** `(equals fref1 fref2) -> boolean`

    Portable Object.equals/Equals

*   **Function** `(instance-of fref type) -> boolean`

    Portable instanceof/is

*   **Function** `(to-string fref) -> string`

    Portable Object.toString/ToString

*   **Function** `(hash fref &key rehash) -> int`

    Portable Object.hasCode/GetHashCode. Note: will cache the value on the fref. If            already cached, will return that, unless :rehash is t.

*   **Function** `(get-type fref) -> Class or Type fref`

    Portable Object.getClass/GetType. Note: will cache the value on the fref. Note also that obtaining the exact type of
    the object is completely independent of the coercion of the fref to its corresponding Lisp type            (see `ensure-typed-ref`)

*   **Function** `(iref indexable-obj &rest indexes) -> a value`

    Calls the default indexer for the object. CLI only. Settable.

<a name="vectors"></a> 

### Vectors

*   **Function** `(make-new-vector type length &rest inits) -> array fref`

    Creates a foreign vector of specified type and length. There can be fewer inits than the length, in which case            the remaining values take the default.

*   **Function** `(vref vector index) -> value`

    Returns the value at the index. Settable.

*   **Function** `(vlength vector index) -> int`

    Returns the length of the vector.

<a name="misc"></a> 

### Miscellaneous

#### Arguments and Boxing

In most cases argument matching and conversion should be transparent. Lisp strings can be passed where Strings are required,
Lisp numbers where int float etc are required. `t` and `nil` can be passed where booleans are required etc. 
`nil` can be passed for `null`.
Some Foil APIs (e.g. make-new-vector) require `type` arguments, and unless specified otherwise, 
any of the following are acceptable:

*   (the value of) A class-symbol - `classname.`
*   A primitive designator keyword - `:boolean|:byte|:char|:double|:float|:int|:long|:short`
*   A fref referring to an actual Class/Type instance
*   A `"package.qualified.ClassName"` string, case-sensitive.                    This is least efficient and should only be used in dynamic scenarios.

    Note that this only applies to Foil APIs, if a foreign runtime API takes a Class/Type argument, you must supply a 
    fref referring to an actual Class/Type instance.

    Occasionally it may be necessary to provide a hint as to the intended type of a numeric argument in order to 
    force resolution to a particular overload.
* **Function** `(box type val)`
    Type must be a primitive designator keyword - `:boolean|:byte|:char|:double|:float|:int|:long|:short`

    Produces an object that when passed to a Foil-generated function will be interpreted by the runtime as that type. 
    Note that silent truncation may occur.


 It is also possible to create vectors in-line as arguments, which will avoid multiple round-trips vs. 
 calling `make-new-vector` in place. Note this is only good for ephemeral vectors, as there is no way to retain a 
 reference to the newly-created vector.

* **Function** `(box-vector type &rest vals)`
    Produces an object that when passed to a Foil-generated function will be an array of type with the supplied values.

    Example:
        CL-USER 102 > (vref (box-vector string. "a" "b" "c" "d") 2) ;only one round-trip
        "c"

#### Class/Type Helpers

*   **Function** `(get-type-for-name full-class-name)`

    Returns a Class/Type instance corresponding to name. Portable Class.forName/Type.GetType.

*   **Function** `(full-class-name class-symbol)`

    Returns the package qualified name string corresponding to the class symbol:

        CL-USER 104 > (full-class-name string.)
        "java.lang.String"

<a name="proxies"></a> 

### Proxies and Callbacks

Proxies allow the creation of foreign objects that implement one or more
interfaces (or in the case of CLI, one or more interfaces or a single delegate)
in Lisp, and thus callbacks from the foreign VM to Lisp. Foil supports Lisp
calling foreign runtime calling Lisp... to an arbitrary (probably
stack-limited) depth.

*   **Generic Function** `(handle-proxy-call method-symbol proxy &rest args))`

 The proxy infrastructure routes all callbacks to this generic function. If a
 proxy p implements an interface i, and the foreign VM ends up invoking i.foo
 on p, it will map to a call to `handle-proxy-call` with the first 2 arguments
 of 'i.foo and p, followed by the actual arguments to the invocation. So,
 callback handlers can be defined by specializing methods on the method-symbol,
 the proxy object, or both. The unspecialized method spews out "unhandled proxy
 call" to standard output and returns nil to the foreign VM. Any unhandled
 errors that occur on the Lisp side during a callback turn into exceptions on
 the runtime server.

*   **Function** `(make-new-proxy arg-marshall-flags arg-marshall-depth &rest interface-types) -> proxy fref`

    Creates and returns a proxy object that implements the given interface
    types or delegate type. `arg-marshall-flags` and `arg-marshall-depth` will
    be used to marshall the arguments to the callback. No handlers are defined
    by this function.

*   **Macro** `(new-proxy proxy arg-marshall-flags arg-marshall-depth &rest interface-defs) -> proxy fref`

    Creates and returns a proxy object that implements the given interface types or delegate.                `

    `proxy -> a symbol`

    `interface-def -> (interface-name method-defs+)`

    `interface-name -> classname.` (must name an interface or delegate type)

    `method-def -> (method-name (args*) body)`

    `method-name -> symbol (without classname)`

    The symbol `proxy` will be bound to the proxy instance in the body of the method implementations.

    Example:
        (new-proxy p +MARSHALL-ID+ 0
               (keylistener.
                 (keyreleased (event)
                   (when (eql *SWT.CR* (keyevent.character event))
                     (gob)))))                </pre>

<a name="marshalling"></a> 

### Marshalling

Foil supports an extensible marshalling system which allows the values of
reference/composite types to be returned in addition to, or even instead of,
the references themselves. Used appropriately, this can substantially reduce
the number of round trips between processes and avoid significant 'chatter'
overhead. 

Marshalling comes into play whenever a reference type is returned from a Foil
function. With certain settings, it is possible to return any or all of a
reference, its hash code, its type, and its value (and the same for any of its
value's reference members), to a specific depth. The nature and depth of the
marshalling is governed by two special variables on the Lisp side -
`*marshalling-flags*` and `*marshalling-depth*`.

The format of marshalled values is determined by the runtime servers, and both
the Java and CLI servers provided with Foil have facilities for adding new
marshallers for specific types. The way an object's value is marshalled is a
function of its type.

Class or Types will always marshall the string representing the
packageQualifiedTypeName, ignoring `*marshalling-depth*`

By default, the following marshalling will be performed when requested, i.e.
`*marshalling-depth*` > 0

*   Arrays marshall as simple Lisp vectors
*   Collections and other enumerable entities marshall as simple Lisp lists
*   Default, if no other marshaller applies - a Lisp assoc-list of keywordized-property-name/value pairs for any public properties
            of the object

*   **Special Variable** `*marshalling-depth*`

Default: 0

A depth of 0 means no values are marshalled, a setting of 1 means that values
will be marshalled for the returned object (if it is a reference), but not any
nested references.  When > 1 nested reference types will marshall, to that
depth of nesting.  

*   **Special Variable** `*marshalling-flags*`

    Either `+MARSHALL-NO-IDS+`, or the logical or-ing of `+MARSHALL-ID+` and
    zero or more of:

    `+MARSHALL-HASH+` and `+MARSHALL-TYPE+`

    Default: `+MARSHALL-ID+`

    A setting of `+MARSHALL-NO-IDS+` means that no frefs will be returned, and
    thus no references will be held on the VM side. If `*marshalling-depth*` is
    0 then nil will be returned. If `*marshalling-depth*` > 0 then the value
    will be returned instead of the fref.

    Otherwise, `+MARSHALL-ID+` must be set, and frefs will be returned for
    reference types.  If `*marshalling-depth*` > 0, then the marshalled values
    will be in the `fref-val` slot. If `+MARSHALL-HASH+` is set then the
    object's hash code will be calculated and stored in the `fref-hash` slot.
    Similarly, if `+MARSHALL-TYPE+` is set then the object's class/type will be
    determined and stored in the `fref-type` slot.  

*   **Macro** `(with-marshalling (depth &rest flags) &body body)`

    Evaluates the body in a context in which the `*marshalling-depth*` is set
    to depth and `*marshalling-flags*` to the `logior` of flags.

*   **Function** `(marshall fref)`

    Explicitly marshalls the object with the current `*marshalling-flags*` and
    `*marshalling-depth*` settings, and returns the marshalled object (which
    may be the same fref, but with additional data in its type/hash/val slots)      

Marshalling example:

    CL-USER 79 > (setf string-class (get-type-for-name "java.lang.String"))
    #}1

    CL-USER 88 > (class.getpackage string-class)
    #}12

    CL-USER 90 > (pprint (with-marshalling (1 +MARSHALL-NO-IDS+)
                           (class.getpackage string-class)))

    ((:IMPLEMENTATIONTITLE . "Java Runtime Environment")
     (:IMPLEMENTATIONVENDOR . "Sun Microsystems, Inc.")
     (:IMPLEMENTATIONVERSION . "1.4.2_05")
     (:NAME . "java.lang")
     (:SEALED)
     (:SPECIFICATIONTITLE . "Java Platform API Specification")
     (:SPECIFICATIONVENDOR . "Sun Microsystems, Inc.")
     (:SPECIFICATIONVERSION . "1.4")) 

<a name="runtimeservers"></a>

## Runtime Servers

Foil includes 2 complete implementations of the runtime server portion of the protocol, one for Java/JVM, the other for C#/CLI. The
implementations are 100% Java/Managed, and use only standard libraries.

Both will run the protocol over one or more TCP sockets specified on the command line, or, if none specified, via standard IO.

Project files are included for Eclipse and Visual Studio. All of the Java code is in com/richhickey/foil and the stand-alone server
is in RuntimeServer.Main. The CLI implementation is in 2 projects, one for the Foil library itself - FoilCLI, and the other for the
stand-alone server - FoilCLISvr.

After building, you can invoke the Java server as follows:

`java -cp . com.richhickey.foil.RuntimeServer 13579`

Make sure the classpath includes the libraries and .jars you will want to use via Foil.

After building, you can invoke the CLI server as follows:

`foilclisvr 13479`

<a name="protocol"></a>

## Protocol

The foil protocol describes the on-stream interface between a Lisp 
instance and a runtime instance, and should not be confused with the foil 
library which provides the interface to the protocol for Common Lisp.  A user
        of Foil will not need to know the protocol, but if you intend to add support
        for another runtime environment (Python anyone?) or host language (Scheme anyone?),
        hopefully this section will help. Note that the protocol docs are not formal, and
        mostly consists of notes to myself and Eric. This will be improved when I get time. For the moment,
        should there be any omissions or inaccuracies here, the Lisp and Java implementations should be 
         considered canonic.

### Connection Services

Foil is a stream-based protocol.  However, no 
protocol is provided for the establishment of the streams - that is an 
implementation detail of the runtime and Lisp libraries.  It is suggested 
that any foil runtime implementation provide at least a stand-alone 
executable server that implements the protocol over its standard IO ports, as well  
        as being able to run over a TCP/IP socket.
Many other scenarios are possible, including multi-socket servers, pre-existing Lisp and runtime 
instances discovering each other etc.  The remainder of the protocol 
description presumes a bi-directional stream has been established.  

Sendable messages:

*   (:cref
*   (:call
*   (:free
*   (:new
*   (:marshall
*   (:hash
*   (:equals
*   (:type-of
*   (:is-a
*   (:str
*   (:tref
*   (:bases
*   (:members
*   (:vector
*   (:vget
*   (:vset
*   (:vlen
*   (:proxy
*   (:iget
*   (:iset

Returnable messages:

*   (:ret
*   (:err
*   (:proxy-call	;only async or from withing a :call

### Invocation Services

Obtaining callable references (crefs)

`(:cref member-type tref|"packageQualifiedTypeName" "memberName")`

Where member-type is an integer representing one of:

*   method (0)
*   field (1)
*   property-get (3)
*   property-set (4)

Note that both Java and the CLI support overloading, so a single member name
might map to multiple overloads.  The resolution of the overloading must occur
in the runtime server at the time of invocation, i.e. any of the overloads may
be called through the same cref.

Returns -> A reference to a callable thing is returned in the standard return format (see below).
`(:ret #{:ref ...})`

####  Creating new object instances

`(:new tref marshall-flags marshall-value-depth-limit (args ...) property-inits ...)`

where property-inits is a set of `:keyword-style-name value` pairs

#### Calling a callable

`(:call cref marshall-flags marshall-value-depth-limit target args ...)`

Example:

`(:call #}101 1 0 2 #}17 "fred")`

Where cref is an cref that has been obtained via :cref, or, only in the case of
calls to Lisp, a symbol that names a function.  marshalling-flags is an integer
representing a bitwise-ORing of:

*   marshall-id (1)
*   marshall-type (2)
*   marshall-hash (4)

a marshall-value-depth-limit of 0 means no reference values are marshalled, a
setting of 1 means that reference values will be marshalled for the return
value (if it is a reference), but not any nested references.  When > 1 nested
reference types will marshall to that depth of nesting.

 If marshalling-flags is 0, no references will be returned (only values) and if
 depth is also 0 then nil will be returned.

target is the object upon which to invoke the method/field/property - pass nil
if static

args are zero or more args as per below.

#### Return Format

one of:

*   `(:ret value)`

    All normal returns are packaged in a form as above, value is as per below.
    If a function has a void return type, nil should be returned.
*   `(:proxy-call ...)`

    A nested callback, in the proxy-call format described below. The receiver
    should process the call, send back its return, then re-read the stream for
    the return value of the original call.
*   `(:err "error description" "stack trace")`

    returned if an exception occurred while processing the request

#### Argument and Return Values

Primitives and Value Types

*   "Strings are in double quotes"
*   Numbers are unadorned decimal numbers with or without a decimal point,leading -, e etc
*   nil is null
*   nil is false
*   t is true


#### Boxed Primitives

Occasionally it may be necessary to provide a hint as to the intended type of a
numeric argument in order to force resolution to a particular overload.

`#{:box typename value}`

Where typename is one of `:byte :int :long :short :float :double`

N.B. silent truncation may occur

Return values should never be boxed

#### vector literals

A vector can be specified in-line as an argument
`#{:vector "packageQualifiedTypeName"|tref|:int(etc) value ...}`


#### References

Reference types are returned with the following tagged syntax:

`#{:ref id rev :type a-ref :hash an-int :val marshalled-value}`

*   :ref, id, and rev must be supplied, all others are optional
*   id - A unique integer reference that identifies the object.
    The object will be kept alive on the hosting side
    until it is freed. Multiple references to the same object will always have
    the same id.
*   :type - A reference to the Type (CLI) or Class (Java)
    object that is the type of the object. Note that this may be the
    first time this reference is seen (and thus it must be registered for
    lifetime maintenance) This will only be available if the marshall-type flag
    is set.
*   :hash - An integer representing the hash value of the object.
    This will only be available if the marshall-hash flag is set.
*   :val - A Lisp-readable representation of the value of the object.
    This will be obtained by using the marshaller registered for the type of
    the object. This will only be available when
    marshall-value-depth-limit is > 0.
    
Note #{ and #} user-space read macros are used by the implementation of foil.
Note that it is possible to return marshalled values of reference objects
without maintaining the reference object on the hosting side (by setting the
marshall-id flag to 0 and having marshall-value-depth-limit > 0

A reference (obtained previously) is passed back to its host like this:

`#}123`

The host will look up the object with that id and pass it along to the call.

#### Exception Reporting

All exceptions are reported via a return of the form:

`(:err "error description" "stack trace")`

if an exception occurred while processing the request. Unless the exception
originated in the reflection API, it is preferred that the stack trace be of
the inner (reflection-invoked) call.

### Object support services

#### Object references

#### Object lifetime management

`(:free refid refrev ...)` -> nil

a list of id/rev pairs is passed.  Allows one or more refs to be GC-ed 
on the hosting side.  It is an error to refer to these refids again.

#### Object marshalling

It is anticipated that runtime servers will provide for user-installable
marshallers, associated with types, that will render the value of an object of
that type on a stream in a form readable by Lisp. By default at least the
following marshallers should be provided:

*   Type|Class - must always marshall the string representing the packageQualifiedTypeName, ignoring marshalling-depth
*   arrays - should marshall as simple vector literals: #(...)
*   collections and other enumerable entities - should marshall as simple lists (...)
*   default, if no other marshaller applies - should yield an assoc-list of keywordized-property-name/value pairs<

In addition to marshalling returns during calls, the value of an object reference can be explicitly marshalled:

`(:marshall ref marshall-flags marshall-value-depth-limit)` -> Lisp-readable-value

Hash values

`(:hash ref)` -> int

Object equality

`(:equals ref ref)` -> t|nil, per Object.Equals

ToString

`(:str ref)` -> "string value"


### Reflection Services

Note, when trefs are returned by these reflection calls, the :val field of the
reference is always (default) marshalled, i.e. set to the
packageQualifiedTypeName as a string.

#### Obtaining a reference to a Type/Class object

`(:tref "packageQualifiedTypeName")` -> tref

#### Object type

`(:type-of ref)` -> tref

`(:is-a ref tref)` -> t|nil

`(:bases tref|"packageQualifiedTypeName")`

    -> (:ret ("packageQualifiedTypeName" ...))    ;most-derived to least-derived

`(:members tref|"packageQualifiedTypeName")`

    -> (:ret ( (:ctors doc-string ...)
            (:methods ((:name string)
                       (:static bool)
                       (:doc doc-string)) ...)
            (:fields ((:name string)
                      (:static bool)
                      (:doc doc-string)) ...)
            (:properties ((:name string)
                          (:static bool)
                          (:get-doc doc-string)
                          (:set-doc doc-string)) ...)))</pre>`

#### Proxies

`(:proxy marshall-flags marshall-value-depth-limit interface-trefs ...)`

    -> (:ret proxy-ref)

Creates a proxy object that implements the given interface(s). When any of the
object's methods are called, sends a :proxy-call message of the form:

`(:proxy-call method-symbol proxy-ref args ...)`

where the proxy-ref is the same one originally returned from the :proxy
message, and the args are marshalled with the flags and depth requested in the
:proxy message. method-symbol has the form

`|package.name|::classname.methodname`

note this means that the Lisp names are not independent, hmmm...

### Vectors

Creating an vector:

*   (:vector tref|"packageQualifiedTypeName" length value ...)
    Creates an vector of the specified type with the specified length Initial
    values are optional and may be fewer than the length.  -> aref
*   (:vget aref marshall-flags marshall-value-depth-limit index)
    -> value
*   (:vset aref index value)
    -> nil
*   (:vlen aref)
    -> int

<a name="summary"></a> 

## Summary

 I'd like to thank my good friend Eric Thorsen for his hard work on the CLI/C# port.

I hope you find Foil useful. It is my sincere intent that it enhance the utility and interoperability of Common Lisp. 
I welcome comments and code contributions.

Rich Hickey, February, 2005


