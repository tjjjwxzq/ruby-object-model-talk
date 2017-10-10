# Types and Structs

All Ruby datatypes like `Object`, `Class`, `Module`, `Array`, `String` etc. are variables in C of type `VALUE`, and `VALUE` is a C typedef that aliases a certain concrete C type (depending on what is available on the platform). This snippet from `include/ruby/ruby.h`:

```c
// include/ruby/ruby.h

#if defined HAVE_UINTPTR_T && 0
typedef uintptr_t VALUE;
typedef uintptr_t ID;
# define SIGNED_VALUE intptr_t
# define SIZEOF_VALUE SIZEOF_UINTPTR_T
# undef PRI_VALUE_PREFIX
#elif SIZEOF_LONG == SIZEOF_VOIDP
typedef unsigned long VALUE;
typedef unsigned long ID;
# define SIGNED_VALUE long
# define SIZEOF_VALUE SIZEOF_LONG
# define PRI_VALUE_PREFIX "l"
#elif SIZEOF_LONG_LONG == SIZEOF_VOIDP
typedef unsigned LONG_LONG VALUE;
typedef unsigned LONG_LONG ID;
# define SIGNED_VALUE LONG_LONG
# define LONG_LONG_VALUE 1
# define SIZEOF_VALUE SIZEOF_LONG_LONG
# define PRI_VALUE_PREFIX PRI_LL_PREFIX
#else
# error ---->> ruby requires sizeof(void*) == sizeof(long) or sizeof(LONG_LONG) to be compiled. <<----
#endif

```
`VALUE` is basically the type of a pointer to the structs that underlie our Ruby datatypes. A generic pointer of type `VALUE` is cast into a pointer to a specific struct with macros such as `RBASIC(obj)` (cast `obj` pointer to point at `RBasic` struct) or `RCLASS(obj)` (cast `obj` pointer to point at `RClass` struct). This snippet from `include/ruby/ruby.h`:

```c
// include/ruby/ruby.h

#define R_CAST(st)   (struct st*)
#define RBASIC(obj)  (R_CAST(RBasic)(obj))
#define ROBJECT(obj) (R_CAST(RObject)(obj))
#define RCLASS(obj)  (R_CAST(RClass)(obj))
#define RMODULE(obj) RCLASS(obj)
#define RSTRING(obj) (R_CAST(RString)(obj))
#define RREGEXP(obj) (R_CAST(RRegexp)(obj))
#define RARRAY(obj)  (R_CAST(RArray)(obj))
#define RDATA(obj)   (R_CAST(RData)(obj))
#define RTYPEDDATA(obj)   (R_CAST(RTypedData)(obj))
#define RFILE(obj)   (R_CAST(RFile)(obj))

```

The structs we will be most interested in are `RBasic`, `RObject`, and `RClass`:

```c
// include/ruby/ruby.h

// Struct types definitions
enum ruby_value_type {
    RUBY_T_NONE   = 0x00,

    RUBY_T_OBJECT = 0x01,
    RUBY_T_CLASS  = 0x02,
    RUBY_T_MODULE = 0x03,
    RUBY_T_FLOAT  = 0x04,
    RUBY_T_STRING = 0x05,
    RUBY_T_REGEXP = 0x06,
    RUBY_T_ARRAY  = 0x07,
    RUBY_T_HASH   = 0x08,
    RUBY_T_STRUCT = 0x09,
    RUBY_T_BIGNUM = 0x0a,
    RUBY_T_FILE   = 0x0b,
    RUBY_T_DATA   = 0x0c,
    RUBY_T_MATCH  = 0x0d,
    RUBY_T_COMPLEX  = 0x0e,
    RUBY_T_RATIONAL = 0x0f,

    RUBY_T_NIL    = 0x11,
    RUBY_T_TRUE   = 0x12,
    RUBY_T_FALSE  = 0x13,
    RUBY_T_SYMBOL = 0x14,
    RUBY_T_FIXNUM = 0x15,
    RUBY_T_UNDEF  = 0x16,

    RUBY_T_IMEMO  = 0x1a,
    RUBY_T_NODE   = 0x1b,
    RUBY_T_ICLASS = 0x1c,
    RUBY_T_ZOMBIE = 0x1d,

    RUBY_T_MASK   = 0x1f
};

#define T_NONE   RUBY_T_NONE
#define T_NIL    RUBY_T_NIL
#define T_OBJECT RUBY_T_OBJECT
#define T_CLASS  RUBY_T_CLASS
#define T_ICLASS RUBY_T_ICLASS
#define T_MODULE RUBY_T_MODULE
#define T_FLOAT  RUBY_T_FLOAT
#define T_STRING RUBY_T_STRING
#define T_REGEXP RUBY_T_REGEXP
#define T_ARRAY  RUBY_T_ARRAY
#define T_HASH   RUBY_T_HASH
#define T_STRUCT RUBY_T_STRUCT
#define T_BIGNUM RUBY_T_BIGNUM
#define T_FILE   RUBY_T_FILE
#define T_FIXNUM RUBY_T_FIXNUM
#define T_TRUE   RUBY_T_TRUE
#define T_FALSE  RUBY_T_FALSE
#define T_DATA   RUBY_T_DATA
#define T_MATCH  RUBY_T_MATCH
#define T_SYMBOL RUBY_T_SYMBOL
#define T_RATIONAL RUBY_T_RATIONAL
#define T_COMPLEX RUBY_T_COMPLEX
#define T_IMEMO  RUBY_T_IMEMO
#define T_UNDEF  RUBY_T_UNDEF
#define T_NODE   RUBY_T_NODE
#define T_ZOMBIE RUBY_T_ZOMBIE
#define T_MASK   RUBY_T_MASK

struct RBasic {
  // stores various 'metadata' about our object
  // most importantly the type of the struct
  // eg. whether its an RObject or RString or RClass etc.
  // (BUILTIN_TYPE(obj) returns something like T_OBJECT)
  // also whether it is a singleton or not (FL_SINGLETON)
  VALUE flags;
  const VALUE klass;
}

// ...

#define ROBJECT_EMBED_LEN_MAX ROBJECT_EMBED_LEN_MAX
#define ROBJECT_EMBED ROBJECT_EMBED
enum {
    ROBJECT_EMBED_LEN_MAX = 3,
    ROBJECT_EMBED = RUBY_FL_USER1, // not sure what RUBY_FL_USERX are or where they are defined

    ROBJECT_ENUM_END
};

struct RObject {
  struct RBasic basic;
  // This union structure allows us to store instance variables embedded
  // directly in the struct (as an array of VALUEs) or in a block of
  // memory pointed to by *ivptr and indexed by the entries in *iv_index_tbl
  // ie. the heap
  // Macros which are used to access the instance variables check the
  // ROBJECT_EMBED flag to check whether the heap of embedded IVs are being
  // used, and fetch the appropriate member in the union

  // since ROBJECT_EMBED_LEN_MAX is set to 3, embedded IV's are used for up to
  // only 3 instance variables
  union {
    struct {
      uint32_t numiv;
      VALUE *ivptr;
      void *iv_index_tbl; /* shortcut for RCLASS_IV_INDEX_TBL(rb_obj_class(obj)) */
    } heap;
    VALUE ary[ROBJECT_EMBED_LEN_MAX];
  } as;
};

// internal.h

struct rb_classext_struct {
    struct st_table *iv_index_tbl;
    struct st_table *iv_tbl;
    struct rb_id_table *const_tbl;
    struct rb_id_table *callable_m_tbl;
    rb_subclass_entry_t *subclasses;
    rb_subclass_entry_t **parent_subclasses;
    /**
     * In the case that this is an `ICLASS`, `module_subclasses` points to the link
     * in the module's `subclasses` list that indicates that the klass has been
     * included. Hopefully that makes sense.
     */
    rb_subclass_entry_t **module_subclasses;
    rb_serial_t class_serial;
    const VALUE origin_;
    VALUE refined_class;
    rb_alloc_func_t allocator;
};

typedef struct rb_classext_struct rb_classext_t;

#undef RClass
struct RClass {
    struct RBasic basic;
    VALUE super;
    rb_classext_t *ptr;
    struct rb_id_table *m_tbl;
};

```

# Class hierarchy
The class hierarchy is initialized in `init_class_hierarchy` in `class.c`:

```c
// class.c

void
Init_class_hierarchy(void)
{
  // note that second argument to `boot_defclass` is super
  rb_cBasicObject = boot_defclass("BasicObject", 0);
  rb_cObject = boot_defclass("Object", rb_cBasicObject);
  rb_gc_register_mark_object(rb_cObject);

  /* resolve class name ASAP for order-independence */
  rb_class_name(rb_cObject);

  rb_cModule = boot_defclass("Module", rb_cObject);
  rb_cClass =  boot_defclass("Class",  rb_cModule);

  rb_const_set(rb_cObject, rb_intern_const("BasicObject"), rb_cBasicObject);
  // Take note here especially
  RBASIC_SET_CLASS(rb_cClass, rb_cClass);
  RBASIC_SET_CLASS(rb_cModule, rb_cClass);
  RBASIC_SET_CLASS(rb_cObject, rb_cClass);
  RBASIC_SET_CLASS(rb_cBasicObject, rb_cClass);
}

```
This in turn is called by `InitVM_Object` in `object.c`:

```c
// object.c
// The method is really long, so I cut it off after the part I care about

void
InitVM_Object(void)
{
    Init_class_hierarchy();

#if 0
    // teach RDoc about these classes
    rb_cBasicObject = rb_define_class("BasicObject", Qnil);
    rb_cObject = rb_define_class("Object", rb_cBasicObject);
    rb_cModule = rb_define_class("Module", rb_cObject);
    rb_cClass =  rb_define_class("Class",  rb_cModule);
#endif

#undef rb_intern
#define rb_intern(str) rb_intern_const(str)

    rb_define_private_method(rb_cBasicObject, "initialize", rb_obj_dummy, 0);
    rb_define_alloc_func(rb_cBasicObject, rb_class_allocate_instance);
    rb_define_method(rb_cBasicObject, "==", rb_obj_equal, 1);
    rb_define_method(rb_cBasicObject, "equal?", rb_obj_equal, 1);
    rb_define_method(rb_cBasicObject, "!", rb_obj_not, 0);
    rb_define_method(rb_cBasicObject, "!=", rb_obj_not_equal, 1);

    rb_define_private_method(rb_cBasicObject, "singleton_method_added", rb_obj_dummy, 1);
    rb_define_private_method(rb_cBasicObject, "singleton_method_removed", rb_obj_dummy, 1);
    rb_define_private_method(rb_cBasicObject, "singleton_method_undefined", rb_obj_dummy, 1);

    /* Document-module: Kernel
     *
     * The Kernel module is included by class Object, so its methods are
     * available in every Ruby object.
     *
     * The Kernel instance methods are documented in class Object while the
     * module methods are documented here.  These methods are called without a
     * receiver and thus can be called in functional form:
     *
     *   sprintf "%.1f", 1.234 #=> "1.2"
     *
     */
    rb_mKernel = rb_define_module("Kernel");
    // Include Kernel in Object
    rb_include_module(rb_cObject, rb_mKernel);
    rb_define_private_method(rb_cClass, "inherited", rb_obj_dummy, 1);
    rb_define_private_method(rb_cModule, "included", rb_obj_dummy, 1);
    rb_define_private_method(rb_cModule, "extended", rb_obj_dummy, 1);
    rb_define_private_method(rb_cModule, "prepended", rb_obj_dummy, 1);
    rb_define_private_method(rb_cModule, "method_added", rb_obj_dummy, 1);
    rb_define_private_method(rb_cModule, "method_removed", rb_obj_dummy, 1);
    rb_define_private_method(rb_cModule, "method_undefined", rb_obj_dummy, 1);

    // ...
    // and more Kernel methods
    // and methods for Module, NilClass, Class etc.
```


# Singleton classes and Metaclasses
Which is which? Are they the same? You can think of a metaclass as a subset of singleton classes. So, in Ruby source, singleton class is used to refer to the singleton class of 'normal' objects. Metaclass is used to refer to the singleton class of 'class' objects (objects which are instances of the `Class` class).

(I use italics to denote the stricter sense of *singleton class* as it is used in Ruby source to differentiate it from the general concept of a singleton class.)

Since singleton classes are themselves instances of `Class`, then the singleton class of a singleton class will be a metaclass. And if the child singleton class is not a *singleton class* but a metaclass, then its singleton class will be a metametaclass. (And of course we can go on and have metametametaclasses, and metametametametaclasses, so in general meta^(n)classes, as they are called in the comments in Ruby source.) The terminology is clarified here:

```c
// class.c

/*!
 * \defgroup class Classes and their hierarchy.
 * \par Terminology
 * - class: same as in Ruby.
 * - singleton class: class for a particular object
 * - eigenclass: = singleton class
 * - metaclass: class of a class. metaclass is a kind of singleton class.
 * - metametaclass: class of a metaclass.
 * - meta^(n)-class: class of a meta^(n-1)-class.
 * - attached object: A singleton class knows its unique instance.
 *   The instance is called the attached object for the singleton class.
 * \{
 */

```

The relevant functions `make_metaclass` and `make_singleton_class`(from `class.c`):

```c
// This will be called with something like
// class A; end
// A.singleton_class

/*!
 * Creates a metaclass of \a klass
 * \param klass     a class
 * \return          created metaclass for the class
 * \pre \a klass is a Class object
 * \pre \a klass has no singleton class.
 * \post the class of \a klass is the returned class.
 * \post the returned class is meta^(n+1)-class when \a klass is a meta^(n)-klass for n >= 0
 */
  static inline VALUE
make_metaclass(VALUE klass)
{
  VALUE super;
  VALUE metaclass = rb_class_boot(Qundef);

  FL_SET(metaclass, FL_SINGLETON);
  rb_singleton_class_attached(metaclass, klass);

  // check if klass is a meta^(n) class of Class class (n can be 0)
  if (META_CLASS_OF_CLASS_CLASS_P(klass)) {
    SET_METACLASS_OF(klass, metaclass);
    SET_METACLASS_OF(metaclass, metaclass);
  }
  else {
    VALUE tmp = METACLASS_OF(klass); /* for a meta^(n)-class klass, tmp is meta^(n)-class of Class class */
    SET_METACLASS_OF(klass, metaclass);
    // NOTE: The ENSURE_EIGENCLASS(klass) returns the metaclass of klass,
    // creating one if it does not already exist
    // hence the above comment
    SET_METACLASS_OF(metaclass, ENSURE_EIGENCLASS(tmp));
  }

  // set the super on the new metaclass
  super = RCLASS_SUPER(klass);
  // this checks to make sure the super isn't an include class (ICLASS)
  // and keeps getting the super until it isn't one
  while (RB_TYPE_P(super, T_ICLASS)) super = RCLASS_SUPER(super);
  // if super exists, set the super of the metaclass to be the metaclass of super
  // otherwise set the super of the metaclass to Class (this happens with BasicObject)
  // so the super of BasicObject's metaclass is Class
  RCLASS_SET_SUPER(metaclass, super ? ENSURE_EIGENCLASS(super) : rb_cClass);

  OBJ_INFECT(metaclass, RCLASS_SUPER(metaclass));

  return metaclass;
}


// This will be called for all non-Class objects
// ie. normal instances and Module objects as well

/*!
 * Creates a singleton class for \a obj.
 * \pre \a obj must not a immediate nor a special const.
 * \pre \a obj must not a Class object.
 * \pre \a obj has no singleton class.
 */
  static inline VALUE
make_singleton_class(VALUE obj)
{
  VALUE orig_class = RBASIC(obj)->klass;
  // rb_class_boot returns a klass whose super is the argument passed to it
  // so klass here has super of orig_class
  VALUE klass = rb_class_boot(orig_class);

  FL_SET(klass, FL_SINGLETON);
  RBASIC_SET_CLASS(obj, klass);
  rb_singleton_class_attached(klass, obj);

  SET_METACLASS_OF(klass, METACLASS_OF(rb_class_real(orig_class)));
  return klass;
}

```

# Method Dispatch
We can get a rough idea of how method dispatch to superclasses work from `vm_eval.c`:

```c
// vm_eval.c

vm_call0_body(rb_thread_t* th, struct rb_calling_info *calling, const struct rb_call_info *ci, struct rb_call_cache *cc, const VALUE *argv)
{
    VALUE ret;

    calling->block_handler = vm_passed_block_handler(th);

  again:
    switch (cc->me->def->type) {
      case VM_METHOD_TYPE_ISEQ:
	{
	    rb_control_frame_t *reg_cfp = th->cfp;
	    int i;

	    CHECK_VM_STACK_OVERFLOW(reg_cfp, calling->argc + 1);

	    *reg_cfp->sp++ = calling->recv;
	    for (i = 0; i < calling->argc; i++) {
		*reg_cfp->sp++ = argv[i];
	    }

	    vm_call_iseq_setup(th, reg_cfp, calling, ci, cc);
	    VM_ENV_FLAGS_SET(th->cfp->ep, VM_FRAME_FLAG_FINISH);
	    return vm_exec(th); /* CHECK_INTS in this function */
	}
      case VM_METHOD_TYPE_NOTIMPLEMENTED:
      case VM_METHOD_TYPE_CFUNC:
	ret = vm_call0_cfunc(th, calling, ci, cc, argv);
	goto success;
      case VM_METHOD_TYPE_ATTRSET:
	rb_check_arity(calling->argc, 1, 1);
	ret = rb_ivar_set(calling->recv, cc->me->def->body.attr.id, argv[0]);
	goto success;
      case VM_METHOD_TYPE_IVAR:
	rb_check_arity(calling->argc, 0, 0);
	ret = rb_attr_get(calling->recv, cc->me->def->body.attr.id);
	goto success;
  // NOTE: For (bound) methods (Method object)
      case VM_METHOD_TYPE_BMETHOD:
	ret = vm_call_bmethod_body(th, calling, ci, cc, argv);
	goto success;
  // NOTE: Here we can see method dispatch in action
  // NOTE: Correction; this is not ordinary method dispatch; that's done in
  // `search_method`. I'm not sure what this is though. Which methods are
  // set as ZSUPER? REFINED probably has to do with refinements
  // Ah, ZSUPER is a call to `super`
      case VM_METHOD_TYPE_ZSUPER:
      case VM_METHOD_TYPE_REFINED:
	{
	    const rb_method_type_t type = cc->me->def->type;
	    VALUE super_class = cc->me->defined_class;

	    if (type == VM_METHOD_TYPE_ZSUPER) {
		super_class = RCLASS_ORIGIN(super_class);
	    }
	    else if (cc->me->def->body.refined.orig_me) {
		cc->me = refined_method_callable_without_refinement(cc->me);
		goto again;
	    }

	    super_class = RCLASS_SUPER(super_class);

      // if we cannot find the superclass, or the method on the super class, we
      // call method_missing instead
	    if (!super_class || !(cc->me = rb_callable_method_entry(super_class, ci->mid))) {
		enum method_missing_reason ex = (type == VM_METHOD_TYPE_ZSUPER) ? MISSING_SUPER : 0;
		ret = method_missing(calling->recv, ci->mid, calling->argc, argv, ex);
		goto success;
	    }
	    RUBY_VM_CHECK_INTS(th);
	    goto again;
	}
      case VM_METHOD_TYPE_ALIAS:
	cc->me = aliased_callable_method_entry(cc->me);
	goto again;
      case VM_METHOD_TYPE_MISSING:
	{
	    vm_passed_block_handler_set(th, calling->block_handler);
	    return method_missing(calling->recv, ci->mid, calling->argc,
				  argv, MISSING_NOENTRY);
	}
      case VM_METHOD_TYPE_OPTIMIZED:
	switch (cc->me->def->body.optimize_type) {
	  case OPTIMIZED_METHOD_TYPE_SEND:
	    ret = send_internal(calling->argc, argv, calling->recv, CALL_FCALL);
	    goto success;
	  case OPTIMIZED_METHOD_TYPE_CALL:
	    {
		rb_proc_t *proc;
		GetProcPtr(calling->recv, proc);
		ret = rb_vm_invoke_proc(th, proc, calling->argc, argv, calling->block_handler);
		goto success;
	    }
	  default:
	    rb_bug("vm_call0: unsupported optimized method type (%d)", cc->me->def->body.optimize_type);
	}
	break;
      case VM_METHOD_TYPE_UNDEF:
	break;
    }
    rb_bug("vm_call0: unsupported method type (%d)", cc->me->def->type);
    return Qundef;

  success:
    RUBY_VM_CHECK_INTS(th);
    return ret;
}

```

(NOTE: This part till the `vm_call0` part is a bit of a rabbit hole. The method type part is however worth noting)
We are most interested in the `VM_METHOD_TYPE_ZSUPER` case, as noted in my added comments. Note the method type the switch statement is checking again (eg. `VM_METHOD_TYPE_ZSUPER`) is taken from the member `cc->me->def->type`. `cc` is a `rb_call_cache` struct, which is defined here:

```c
// vm_core.c

struct rb_call_cache {
    /* inline cache: keys */
    rb_serial_t method_state;
    rb_serial_t class_serial;

    /* inline cache: values */
    const rb_callable_method_entry_t *me;

    vm_call_handler call;

    union {
	unsigned int index; /* used by ivar */
	enum method_missing_reason method_missing_reason; /* used by method_missing */
	int inc_sp; /* used by cfunc */
    } aux;
};

```

`me` is an `rb_callable_method_entry` struct, defined in `method.h`

```c
// method.h

typedef struct rb_callable_method_entry_struct { /* same fields with rb_method_entry_t */
    VALUE flags;
    const VALUE defined_class;
    struct rb_method_definition_struct * const def;
    ID called_id;
    const VALUE owner;
} rb_callable_method_entry_t;

```

We see that the `type` member of the `rb_method_entry_struct` is an enum that can take the following values:

```c
// method.h

typedef enum {
    // normal methods we define in Ruby with `def`
    VM_METHOD_TYPE_ISEQ,
    // Ruby methods defined from C, with `rb_define_method`
    VM_METHOD_TYPE_CFUNC,
    // Ruby attribute setter method defined with `attr_writer`/`attr_accesor`
    VM_METHOD_TYPE_ATTRSET,
    // Ruby attribute getter method defined with `attr_reader`/`attr_accesor`
    VM_METHOD_TYPE_IVAR,
    // Bound method objects (`Method`)
    VM_METHOD_TYPE_BMETHOD,
    // Call to `super`
    VM_METHOD_TYPE_ZSUPER,
    // Aliased methods
    VM_METHOD_TYPE_ALIAS,
    // Undefined method (using `undef`)
    VM_METHOD_TYPE_UNDEF,
    // For methods not implemented on the current platform
    VM_METHOD_TYPE_NOTIMPLEMENTED,
    VM_METHOD_TYPE_OPTIMIZED, /* Kernel#send, Proc#call, etc */
    VM_METHOD_TYPE_MISSING,   /* wrapper for method_missing(id) */
    // Refinements
    VM_METHOD_TYPE_REFINED,

    END_OF_ENUMERATION(VM_METHOD_TYPE)
} rb_method_type_t;

```

Now the question is how the method types are set when the methods are defined. We see in `class.c` the way to define Ruby methods from C:

```c

// class.c

void
rb_define_method_id(VALUE klass, ID mid, VALUE (*func)(ANYARGS), int argc)
{
    rb_add_method_cfunc(klass, mid, func, argc, METHOD_VISI_PUBLIC);
}

void
rb_define_method(VALUE klass, const char *name, VALUE (*func)(ANYARGS), int argc)
{
    rb_add_method_cfunc(klass, rb_intern(name), func, argc, METHOD_VISI_PUBLIC);
}

void
rb_define_protected_method(VALUE klass, const char *name, VALUE (*func)(ANYARGS), int argc)
{
    rb_add_method_cfunc(klass, rb_intern(name), func, argc, METHOD_VISI_PROTECTED);
}

void
rb_define_private_method(VALUE klass, const char *name, VALUE (*func)(ANYARGS), int argc)
{
    rb_add_method_cfunc(klass, rb_intern(name), func, argc, METHOD_VISI_PRIVATE);
}

```

but you can see that all these are calling `rb_add_method_cfunc`, which sets the method as a `VM_METHOD_TYPE_CFUNC`, so the case in the switch statement of `vm_call0_body` does not involve anything to do with method dispatch to super classes (makes sense, since the methods defined directly in C code are defined on the object itself).

However, let's look again at `vm_call0_body`, and find out where it's called, to see if we can figure out where the method entry is coming from. It's called from `vm_call0`, also in `vm_eval.c`:

```c
// vm_eval.c

static VALUE
vm_call0(rb_thread_t* th, VALUE recv, ID id, int argc, const VALUE *argv, const rb_callable_method_entry_t *me)
{
    struct rb_calling_info calling_entry, *calling;
    struct rb_call_info ci_entry;
    struct rb_call_cache cc_entry;

    calling = &calling_entry;

    ci_entry.flag = 0;
    ci_entry.mid = id;

    cc_entry.me = me;

    calling->recv = recv;
    calling->argc = argc;

    return vm_call0_body(th, calling, &ci_entry, &cc_entry, argv);
}

```

`vm_call0` is taking the method entry as an argument, so where then is `vm_call0` being called? In several different functions below its definition, but let's look at `rb_call0` (looks the most promising):

```c

/*!
 * \internal
 * calls the specified method.
 *
 * This function is called by functions in rb_call* family.
 * \param recv   receiver of the method
 * \param mid    an ID that represents the name of the method
 * \param argc   the number of method arguments
 * \param argv   a pointer to an array of method arguments
 * \param scope
 * \param self   self in the caller. Qundef means no self is considered and
 *               protected methods cannot be called
 *
 * \note \a self is used in order to controlling access to protected methods.
 */
static inline VALUE
rb_call0(VALUE recv, ID mid, int argc, const VALUE *argv,
	 call_type scope, VALUE self)
{
    // we are interested in this!
    const rb_callable_method_entry_t *me = rb_search_method_entry(recv, mid);
    rb_thread_t *th = GET_THREAD();
    enum method_missing_reason call_status = rb_method_call_status(th, me, scope, self);

    if (call_status != MISSING_NONE) {
	return method_missing(recv, mid, argc, argv, call_status);
    }
    stack_check(th);
    return vm_call0(th, recv, mid, argc, argv, me);
}

```

So it basically searches up the method entry from the method ID and the receiver struct with `rb_search_method_entry`. Let's see what that method is up to:

```c
// vm_eval.c

static inline const rb_callable_method_entry_t *
rb_search_method_entry(VALUE recv, ID mid)
{
    VALUE klass = CLASS_OF(recv);

    // NOTE: This branch is all error-checking;
    // which conceptually we don't need to worry about
    if (!klass) {
        VALUE flags;
        if (SPECIAL_CONST_P(recv)) {
            rb_raise(rb_eNotImpError,
                     "method `%"PRIsVALUE"' called on unexpected immediate object (%p)",
                     rb_id2str(mid), (void *)recv);
        }
        flags = RBASIC(recv)->flags;
        if (flags == 0) {
            rb_raise(rb_eNotImpError,
                     "method `%"PRIsVALUE"' called on terminated object"
                     " (%p flags=0x%"PRIxVALUE")",
                     rb_id2str(mid), (void *)recv, flags);
        }
        else {
            int type = BUILTIN_TYPE(recv);
            const char *typestr = rb_type_str(type);
            if (typestr && T_OBJECT <= type && type < T_NIL)
                rb_raise(rb_eNotImpError,
                         "method `%"PRIsVALUE"' called on hidden %s object"
                         " (%p flags=0x%"PRIxVALUE")",
                         rb_id2str(mid), typestr, (void *)recv, flags);
            if (typestr)
                rb_raise(rb_eNotImpError,
                         "method `%"PRIsVALUE"' called on unexpected %s object"
                         " (%p flags=0x%"PRIxVALUE")",
                         rb_id2str(mid), typestr, (void *)recv, flags);
            else
                rb_raise(rb_eNotImpError,
                         "method `%"PRIsVALUE"' called on broken T_???" "(0x%02x) object"
                         " (%p flags=0x%"PRIxVALUE")",
                         rb_id2str(mid), type, (void *)recv, flags);
        }
    }
    // NOTE: This is the meat of it
    return rb_callable_method_entry(klass, mid);
}

```

It looks like a lot going on, but you realize that the big `if` block actually just does a whole bunch of error-checking that is conceptually uninteresting. We should look at `rb_callable_method_entry` (in `vm_method.c`) instead, since that's what's doing the real work:

```c

// vm_method.c

const rb_callable_method_entry_t *
rb_callable_method_entry(VALUE klass, ID id)
{
    VALUE defined_class;
    // this is where our method entry comes from!
    rb_method_entry_t *me = method_entry_get(klass, id, &defined_class);
    return prepare_callable_method_entry(defined_class, id, me);
}

```

Chase the delegation chain down further, to `method_entry_get`:

```c

// vm_method.c

static rb_method_entry_t *
method_entry_get(VALUE klass, ID id, VALUE *defined_class_ptr)
{
#if OPT_GLOBAL_METHOD_CACHE
    struct cache_entry *ent;
    ent = GLOBAL_METHOD_CACHE(klass, id);
    if (ent->method_state == GET_GLOBAL_METHOD_STATE() &&
	ent->class_serial == RCLASS_SERIAL(klass) &&
	ent->mid == id) {
#if VM_DEBUG_VERIFY_METHOD_CACHE
	verify_method_cache(klass, id, ent->defined_class, ent->me);
#endif
	if (defined_class_ptr) *defined_class_ptr = ent->defined_class;
	return ent->me;
    }
#endif

    return method_entry_get_without_cache(klass, id, defined_class_ptr);
}

```

Still not where we can see method dispatch happening, but we're almost there. The bulk of `method_entry_get` is dealing with fetching the method entry from the global method cache, if it's being used. Let's look at `method_entry_get_without_cache`:

```c
// vm_method.c

/*
 * search method entry without the method cache.
 *
 * if you need method entry with method cache (normal case), use
 * rb_method_entry() simply.
 */
static rb_method_entry_t *
method_entry_get_without_cache(VALUE klass, ID id,
			       VALUE *defined_class_ptr)
{
    VALUE defined_class;
    // This is what we care about
    rb_method_entry_t *me = search_method(klass, id, &defined_class);

    if (ruby_running) {
	if (OPT_GLOBAL_METHOD_CACHE) {
	    struct cache_entry *ent;
	    ent = GLOBAL_METHOD_CACHE(klass, id);
	    ent->class_serial = RCLASS_SERIAL(klass);
	    ent->method_state = GET_GLOBAL_METHOD_STATE();
	    ent->defined_class = defined_class;
	    ent->mid = id;

	    if (UNDEFINED_METHOD_ENTRY_P(me)) {
		me = ent->me = NULL;
	    }
	    else {
		ent->me = me;
	    }
	}
	else if (UNDEFINED_METHOD_ENTRY_P(me)) {
	    me = NULL;
	}
    }
    else if (UNDEFINED_METHOD_ENTRY_P(me)) {
	me = NULL;
    }

    if (defined_class_ptr)
	*defined_class_ptr = defined_class;
    return me;
}

```

Again, the crucial bit is on a single line, with the invocation of `search_method`. The rest is just caching the method entry and error checking. So one last method jump:

```c
// vm_method.c


static inline rb_method_entry_t*
search_method(VALUE klass, ID id, VALUE *defined_class_ptr)
{
    rb_method_entry_t *me;

    // This is what we were looking for
    for (me = 0; klass; klass = RCLASS_SUPER(klass)) {
	if ((me = lookup_method_table(klass, id)) != 0) break;
    }

    // If we are passed a valid pointer, set it to
    // the class in which the method was defined too
    if (defined_class_ptr)
	*defined_class_ptr = klass;
    return me;
}

```

And the crux of it is just the simple `for` loop which sets `klass` to the superclass of the current class each iteration, and tries to look up the method in the current class. If we find a valid method entry, we break out of the loop. If not, the loop terminates if `klass` (and hence `RCLASS_SUPER(klass)` from the previous iteration) evaluates to `NULL`; that is, we've reached the end of the inheritance chain. Then `me` is just `0`, and that value is returned. Jumping back to `method_entry_get_without_cache`, we see that if `me` is `0`, we will enter the `UNDEFINED_METHOD_ENTRY_P(me)` branch, and `me` will be set to `NULL`.

Let's look at what's happening in `lookup_method_table` in more detail (`vm_method.c`):

```c
// vm_method.c

static inline rb_method_entry_t *
lookup_method_table(VALUE klass, ID id)
{
    st_data_t body;
    // get the method table from RClass struct
    struct rb_id_table *m_tbl = RCLASS_M_TBL(klass);

    // read the method entry into st_data_t body
    if (rb_id_table_lookup(m_tbl, id, &body)) {
	return (rb_method_entry_t *) body;
    }
    else {
	return 0;
    }
}

```

# Defining new classes

When we define a class at the top-level (under the `Object` namespace), `rb_define_class` in `class.c` is called (there is also `rb_define_class_under` when defining a class under another namespace, which is more or less similar):

```c
// class.c

/*!
 * Defines a top-level class.
 * \param name   name of the class
 * \param super  a class from which the new class will derive.
 * \return the created class
 * \throw TypeError if the constant name \a name is already taken but
 *                  the constant is not a \c Class.
 * \throw TypeError if the class is already defined but the class can not
 *                  be reopened because its superclass is not \a super.
 * \throw ArgumentError if the \a super is NULL.
 * \post top-level constant named \a name refers the returned class.
 *
 * \note if a class named \a name is already defined and its superclass is
 *       \a super, the function just returns the defined class.
 */
VALUE
rb_define_class(const char *name, VALUE super)
{
    VALUE klass;
    ID id;

    id = rb_intern(name);
    if (rb_const_defined(rb_cObject, id)) {
	klass = rb_const_get(rb_cObject, id);
	if (!RB_TYPE_P(klass, T_CLASS)) {
	    rb_raise(rb_eTypeError, "%s is not a class (%"PRIsVALUE")",
		     name, rb_obj_class(klass));
	}
	if (rb_class_real(RCLASS_SUPER(klass)) != super) {
	    rb_raise(rb_eTypeError, "superclass mismatch for class %s", name);
	}
	return klass;
    }
    // this method throws if super is not defined. I suppose if we define a class
    // at the top level with no explicit superclass, `rb_define_class_id` is called
    // directly instead of this? Because there, if super is NULL, it is set to Object
    if (!super) {
	rb_raise(rb_eArgError, "no super class for `%s'", name);
    }
    klass = rb_define_class_id(id, super);
    rb_vm_add_root_module(id, klass);
    rb_name_class(klass, id);
    rb_const_set(rb_cObject, id, klass);
    rb_class_inherited(super, klass);

    return klass;
}
```

Our `klass` is defined with `rb_define_class_id`:

```c
// class.c

/*!
 * Defines a new class.
 * \param id     ignored
 * \param super  A class from which the new class will derive. NULL means \c Object class.
 * \return       the created class
 * \throw TypeError if super is not a \c Class object.
 *
 * \note the returned class will not be associated with \a id.
 *       You must explicitly set a class name if necessary.
 */
VALUE
rb_define_class_id(ID id, VALUE super)
{
    VALUE klass;

    // default super is Object
    if (!super) super = rb_cObject;
    // creates the class with the given super
    klass = rb_class_new(super);
    // we make the metaclass straightaway
    // note that the second argument is actually unused
    rb_make_metaclass(klass, RBASIC(super)->klass);

    return klass;
}

```

Now`rb_class_new`:

```c
// class.c

/*!
 * Creates a new class.
 * \param super     a class from which the new class derives.
 * \exception TypeError \a super is not inheritable.
 * \exception TypeError \a super is the Class class.
 */
VALUE
rb_class_new(VALUE super)
{
    // make sure super is a Class class
    Check_Type(super, T_CLASS);
    // make sure super can be inherited from
    rb_check_inheritable(super);
    // create the new class
    return rb_class_boot(super);
}

```

And `rb_class_boot`:

```c
// class.c

/*!
 * A utility function that wraps class_alloc.
 *
 * allocates a class and initializes safely.
 * \param super     a class from which the new class derives.
 * \return          a class object.
 * \pre  \a super must be a class.
 * \post the metaclass of the new class is Class.
 */
VALUE
rb_class_boot(VALUE super)
{
    VALUE klass = class_alloc(T_CLASS, rb_cClass);

    RCLASS_SET_SUPER(klass, super);
    RCLASS_M_TBL_INIT(klass);

    OBJ_INFECT(klass, super);
    return (VALUE)klass;
}

```

And `rb_make_metaclass`:

```c
// class.c

/*!
 * \internal
 * Creates a new *singleton class* for an object.
 *
 * \pre \a obj has no singleton class.
 * \note DO NOT USE the function in an extension libraries. Use \ref rb_singleton_class.
 * \param obj     An object.
 * \param unused  ignored.
 * \return        The singleton class of the object.
 */
VALUE
rb_make_metaclass(VALUE obj, VALUE unused)
{
    if (BUILTIN_TYPE(obj) == T_CLASS) {
	return make_metaclass(obj);
    }
    else {
	return make_singleton_class(obj);
    }
}

Of course, since we're talking about creating classes, we end up calling `make_metaclass`, which is where the fun stuff happens:

```c
// class.c

/*!
 * Creates a metaclass of \a klass
 * \param klass     a class
 * \return          created metaclass for the class
 * \pre \a klass is a Class object
 * \pre \a klass has no singleton class.
 * \post the class of \a klass is the returned class.
 * \post the returned class is meta^(n+1)-class when \a klass is a meta^(n)-klass for n >= 0
 */
static inline VALUE
make_metaclass(VALUE klass)
{
    VALUE super;
    VALUE metaclass = rb_class_boot(Qundef);

    FL_SET(metaclass, FL_SINGLETON);
    rb_singleton_class_attached(metaclass, klass);

    if (META_CLASS_OF_CLASS_CLASS_P(klass)) {
	SET_METACLASS_OF(klass, metaclass);
	SET_METACLASS_OF(metaclass, metaclass);
    }
    else {
	VALUE tmp = METACLASS_OF(klass); /* for a meta^(n)-class klass, tmp is meta^(n)-class of Class class */
	SET_METACLASS_OF(klass, metaclass);
	SET_METACLASS_OF(metaclass, ENSURE_EIGENCLASS(tmp));
    }

    super = RCLASS_SUPER(klass);
    while (RB_TYPE_P(super, T_ICLASS)) super = RCLASS_SUPER(super);
    RCLASS_SET_SUPER(metaclass, super ? ENSURE_EIGENCLASS(super) : rb_cClass);

    OBJ_INFECT(metaclass, RCLASS_SUPER(metaclass));

    return metaclass;
}

```

See the comments in the Singleton Classes section on what this method does. But, it's basically four things: 1) Initialize the metaclass (new RClass struct), setting its singleton flag. 2) Setting the metaclass of our class (klass) to the new RClass struct we just created. 3) Setting the metaclass of our newly created metaclass. This spawns a whole chain reaction where we have to fetch/create the metaclass of Class (we end up calling `make_metaclass(Class)`), which requires us setting the metaclass and superclass of the metaclass of Class. So we end up creating the metaclass of Module to be the superclass to the metaclass of Class. And then we need the metaclass of Object to be the superclass to the metaclass of Module. And so on. And the superclass of BasicObject's metaclass is Class itself. Step 3) is a lot of work. 4) Finally, set the superclass of our new metaclass. If the superclass of our class was Object, then this turns out to be the metaclass of Object. Otherwise it's the metaclass of the superclass of our class.

# Defining Modules
Similar to classes, we have `rb_define_module_id`:

```c
// class.c

VALUE
rb_define_module_id(ID id)
{
    VALUE mdl;

    mdl = rb_module_new();
    rb_name_class(mdl, id);

    return mdl;
}

```

It's much simpler than classes it seems. Looking at `rb_module_new()`:

```c
// class.c

VALUE
rb_module_new(void)
{
    VALUE mdl = class_alloc(T_MODULE, rb_cModule);
    RCLASS_M_TBL_INIT(mdl);
    return (VALUE)mdl;
}

```

Just like classes, we also initialize the module with `class_alloc`, because it's also stored as an `RClass` struct. But we pass it a `T_MODULE` type this time, instead of a `T_CLASS`. And of course its `klass` should be `Module`. Also, like classes, we initialize the method table, but unlike classes, *we do not set super*. Module's aren't supposed to have superclasses! Not from the Ruby side, at least (`Module` does not have a `superclass` instance method). But in reality, we can set the `super` of a module's `RClass` struct, if we include another module within it; then `super` will point to the include class of the included module.

# Including Modules (Include Classes)
Look at `rb_include_module`:

```c
// class.c

void
rb_include_module(VALUE klass, VALUE module)
{
    int changed = 0;

    rb_frozen_class_p(klass);
    Check_Type(module, T_MODULE);
    OBJ_INFECT(klass, module);

    changed = include_modules_at(klass, RCLASS_ORIGIN(klass), module, TRUE);
    if (changed < 0)
	rb_raise(rb_eArgError, "cyclic include detected");
}

```

Most of the crazy stuff happens in `include_modules_at`. (Also, `RCLASS_ORIGIN` is basically set to `klass` itself for a fresh class that does not have any modules included yet; ie. when first initialized):

```c
// class.c

static int
include_modules_at(const VALUE klass, VALUE c, VALUE module, int search_super)
{
    VALUE p, iclass;
    int method_changed = 0, constant_changed = 0;
    struct rb_id_table *const klass_m_tbl = RCLASS_M_TBL(RCLASS_ORIGIN(klass));

    while (module) {
	int superclass_seen = FALSE;
	struct rb_id_table *tbl;

  // remember that a freshly initialized module (or class) has the origin
  // field set to itself. So I don't know when this would be true
	if (RCLASS_ORIGIN(module) != module)
	    goto skip;
  // this will happen if you try to include module in itself
  // ie. a cyclic include
  // or eg. you include M2 in M1, then try to include M1 in M2
	if (klass_m_tbl && klass_m_tbl == RCLASS_M_TBL(module))
	    return -1;
	/* ignore if the module included already in superclasses */
	for (p = RCLASS_SUPER(klass); p; p = RCLASS_SUPER(p)) {
	    int type = BUILTIN_TYPE(p);
	    if (type == T_ICLASS) {
		if (RCLASS_M_TBL(p) == RCLASS_M_TBL(module)) {
		    if (!superclass_seen) {
			c = p;  /* move insertion point */
		    }
		    goto skip;
		}
	    }
	    else if (type == T_CLASS) {
		if (!search_super) break;
		superclass_seen = TRUE;
	    }
	}
  // in the normal case, c, the insertion point, will not have changed
  // and will be klass itself
	iclass = rb_include_class_new(module, RCLASS_SUPER(c));
	c = RCLASS_SET_SUPER(c, iclass);

  // included in this module
	{
	    VALUE m = module;
      // this will be true if the while(module) loop executes
      // more than once, because the only `super` a module
      // can have is an include class (see the skip label)

      // the klass of our include class basically points to its original
      // module, so we set m to that
      // and we add it to the subclasses of the include class of the
      // module we are currently including
	    if (BUILTIN_TYPE(m) == T_ICLASS) m = RBASIC(m)->klass;
	    rb_module_add_to_subclasses_list(m, iclass);
	}

	if (FL_TEST(klass, RMODULE_IS_REFINEMENT)) {
	    VALUE refined_class =
		rb_refinement_module_get_refined_class(klass);

	    rb_id_table_foreach(RMODULE_M_TBL(module), add_refined_method_entry_i, (void *)refined_class);
	    FL_SET(c, RMODULE_INCLUDED_INTO_REFINEMENT);
	}

	tbl = RMODULE_M_TBL(module);
	if (tbl && rb_id_table_size(tbl)) method_changed = 1;

	tbl = RMODULE_CONST_TBL(module);
	if (tbl && rb_id_table_size(tbl)) constant_changed = 1;
      skip:
      // if our module doesn't include any other modules, this will be null
      // and we will break out of the while loop
      // otherwise module will now be the include class of the module included
      // in our module.
	module = RCLASS_SUPER(module);
    }

    if (method_changed) rb_clear_method_cache_by_class(klass);
    if (constant_changed) rb_clear_constant_cache();

    return method_changed;
}


```

Let's peek at `rb_include_class_new`:

```c
// class.c

VALUE
rb_include_class_new(VALUE module, VALUE super)
{
    VALUE klass = class_alloc(T_ICLASS, rb_cClass);

    // this will happen if we include a module that has another
    // module included in it
    if (BUILTIN_TYPE(module) == T_ICLASS) {
	module = RBASIC(module)->klass;
    }
    if (!RCLASS_IV_TBL(module)) {
	RCLASS_IV_TBL(module) = st_init_numtable();
    }
    if (!RCLASS_CONST_TBL(module)) {
	RCLASS_CONST_TBL(module) = rb_id_table_create(0);
    }
    RCLASS_IV_TBL(klass) = RCLASS_IV_TBL(module);
    RCLASS_CONST_TBL(klass) = RCLASS_CONST_TBL(module);

    RCLASS_M_TBL(OBJ_WB_UNPROTECT(klass)) =
      RCLASS_M_TBL(OBJ_WB_UNPROTECT(RCLASS_ORIGIN(module))); /* TODO: unprotected? */

    RCLASS_SET_SUPER(klass, super);
    if (RB_TYPE_P(module, T_ICLASS)) {
	RBASIC_SET_CLASS(klass, RBASIC(module)->klass);
    }
    else {
	RBASIC_SET_CLASS(klass, module);
    }
    OBJ_INFECT(klass, module);
    OBJ_INFECT(klass, super);

    return (VALUE)klass;
}

```


Prepending modules is not as straightforward as I might have imagined it. Instead of directly inserting the module include class below the current class, a duplicate of the current class is made (and assigned to `origin` as seen below). This duplicate actually has an include class type, and copies the method table of the current class (`klass`). Then, the method table of the current class is cleared (`RCLASS_M_TBL_INIT(klass)`). Finally, we still insert the module as normal *above* our current class. But since its method table is empty, it is as good as not there, as far as method dispatched is concerned. So our duplicated include class now stands in as the proxy of the class to which we prepended out module.

```c
// class.c

void
rb_prepend_module(VALUE klass, VALUE module)
{
    VALUE origin;
    int changed = 0;

    rb_frozen_class_p(klass);
    Check_Type(module, T_MODULE);
    OBJ_INFECT(klass, module);

    origin = RCLASS_ORIGIN(klass);
    if (origin == klass) {
	origin = class_alloc(T_ICLASS, klass);
	OBJ_WB_UNPROTECT(origin); /* TODO: conservative shading. Need more survey. */
	RCLASS_SET_SUPER(origin, RCLASS_SUPER(klass));
	RCLASS_SET_SUPER(klass, origin);
	RCLASS_SET_ORIGIN(klass, origin);
	RCLASS_M_TBL(origin) = RCLASS_M_TBL(klass);
	RCLASS_M_TBL_INIT(klass);
	rb_id_table_foreach(RCLASS_M_TBL(origin), move_refined_method, (void *)klass);
    }
    changed = include_modules_at(klass, klass, module, FALSE);
    if (changed < 0)
	rb_raise(rb_eArgError, "cyclic prepend detected");
    if (changed) {
	rb_vm_check_redefinition_by_prepend(klass);
    }
}

```

# Object Instantiation
This happens in `rb_class_new_instance` in `object.c` (don't confuse it with the `Class::new` class method of `Class`, which creates a new class object, while something like `Dog.new` is an instance method `Class#new`):

```c
// object.c

/*
 *  call-seq:
 *     class.new(args, ...)    ->  obj
 *
 *  Calls <code>allocate</code> to create a new object of
 *  <i>class</i>'s class, then invokes that object's
 *  <code>initialize</code> method, passing it <i>args</i>.
 *  This is the method that ends up getting called whenever
 *  an object is constructed using .new.
 *
 */

VALUE
rb_class_new_instance(int argc, const VALUE *argv, VALUE klass)
{
    VALUE obj;

    // allocates space for object without calling initialize
    obj = rb_obj_alloc(klass);
    // invokes `initialize` method, passing requisite args
    rb_obj_call_init(obj, argc, argv);

    return obj;
}

```

# Instance Variables
We can see how instance variables are looked up when we look at the corresponding C method to `instance_variable_get`, which is `rb_obj_ivar_get` in `object.c`:

```c
// object.c

static VALUE
rb_obj_ivar_get(VALUE obj, VALUE iv)
{
    ID id = id_for_var(obj, iv, an, instance);

    if (!id) {
	return Qnil;
    }
    return rb_ivar_get(obj, id);
}

```

Let's follow `rb_ivar_get` in `variable.c`:

```c
// variable.c

VALUE
rb_ivar_get(VALUE obj, ID id)
{
    VALUE iv = rb_ivar_lookup(obj, id, Qundef);

    if (iv == Qundef) {
	if (RTEST(ruby_verbose))
	    rb_warning("instance variable %"PRIsVALUE" not initialized", QUOTE_ID(id));
	iv = Qnil;
    }
    return iv;
}

```

Clearly the bulk of the work is in `rb_ivar_lookup`, so one more jump:

```c
// variable.c

VALUE
rb_ivar_lookup(VALUE obj, ID id, VALUE undef)
{
    VALUE val, *ptr;
    struct st_table *iv_index_tbl;
    uint32_t len;
    st_data_t index;

    if (SPECIAL_CONST_P(obj)) return undef;
    switch (BUILTIN_TYPE(obj)) {
      // Normal objects
      case T_OBJECT:
        len = ROBJECT_NUMIV(obj);
        ptr = ROBJECT_IVPTR(obj);
        // NOTE: for instances of classes, we get the RObject.iv_index_tbl
        // member, which is a pointer to the same struct that is pointed
        // to by the `struct st_table *iv_index_tbl` in the `rb_classext_struct`
        iv_index_tbl = ROBJECT_IV_INDEX_TBL(obj);
        if (!iv_index_tbl) break;
        // st_lookup reads the value from iv_index_tbl at the key id into
        // the memory pointed to by index
        if (!st_lookup(iv_index_tbl, (st_data_t)id, &index)) break;
        if (len <= index) break;
        // ptr = ivptr points to the block of memory where the actual
        // instance variable values are stored. What is stored in iv_index_tbl
        // is just the offset, or index, hence its name (d'oh)
        val = ptr[index];
        if (val != Qundef)
            return val;
	break;
      // Classes and modules
      case T_CLASS:
      case T_MODULE:
      // NOTE: for classes and modules, we get the rb_classext_struct.iv_tbl
      // member, vs. the iv_index_tbl member for instances
	if (RCLASS_IV_TBL(obj) &&
		st_lookup(RCLASS_IV_TBL(obj), (st_data_t)id, &index))
	    return (VALUE)index;
	break;
      default:
	if (FL_TEST(obj, FL_EXIVAR))
	    return generic_ivar_get(obj, id, undef);
	break;
    }
    return undef;
}

```

What about setting variables? Let's turn to `rb_ivar_set` in `variable.c`:

```c
// variable.c

VALUE
rb_ivar_set(VALUE obj, ID id, VALUE val)
{
    struct ivar_update ivup;
    uint32_t i, len;

    rb_check_frozen(obj);

    switch (BUILTIN_TYPE(obj)) {
      case T_OBJECT:
        ivup.iv_extended = 0;
        ivup.u.iv_index_tbl = iv_index_tbl_make(obj);
        // This adds the new instance variable *index* entry to the iv_index_table
        iv_index_tbl_extend(&ivup, id);
        len = ROBJECT_NUMIV(obj);
        // NOTE: This big `if` block is for checking if the size of our
        // updated iv_index_table is larger than the numiv value stored
        // on our `obj` struct (ie. it has overflowed), and reallocating
        // the numiv and ivptr members to accomodate the new size
        if (len <= ivup.index) {
            VALUE *ptr = ROBJECT_IVPTR(obj);
            if (ivup.index < ROBJECT_EMBED_LEN_MAX) {
                RBASIC(obj)->flags |= ROBJECT_EMBED;
                ptr = ROBJECT(obj)->as.ary;
                for (i = 0; i < ROBJECT_EMBED_LEN_MAX; i++) {
                    ptr[i] = Qundef;
                }
            }
            else {
                VALUE *newptr;
                uint32_t newsize = iv_index_tbl_newsize(&ivup);

                if (RBASIC(obj)->flags & ROBJECT_EMBED) {
                    newptr = ALLOC_N(VALUE, newsize);
                    MEMCPY(newptr, ptr, VALUE, len);
                    RBASIC(obj)->flags &= ~ROBJECT_EMBED;
                    ROBJECT(obj)->as.heap.ivptr = newptr;
                }
                else {
                    REALLOC_N(ROBJECT(obj)->as.heap.ivptr, VALUE, newsize);
                    newptr = ROBJECT(obj)->as.heap.ivptr;
                }
                for (; len < newsize; len++)
                    newptr[len] = Qundef;
                ROBJECT(obj)->as.heap.numiv = newsize;
                ROBJECT(obj)->as.heap.iv_index_tbl = ivup.u.iv_index_tbl;
            }
        }
        // And this is where we write the actual instance variable value
        // RB_OBJ_WRITE(a, slot, b)
        // Writes b into *slot
        // where slot is a pointer in a
        RB_OBJ_WRITE(obj, &ROBJECT_IVPTR(obj)[ivup.index], val);
	break;
      case T_CLASS:
      case T_MODULE:
	if (!RCLASS_IV_TBL(obj)) RCLASS_IV_TBL(obj) = st_init_numtable();
	rb_class_ivar_set(obj, id, val);
        break;
      default:
	generic_ivar_set(obj, id, val);
	break;
    }
    return val;
}
```

It seems to use some kind of `ivar_update` struct to actually add the instance variable in. The `ivar_update` struct sets its `iv_index_tbl` pointer member to the same `st_table` as is stored on the `obj` (if it is a normal object; a new `st_table` is initialized if it does not already exist). Then we extend the `iv_index_tbl`:

```c
// variable.c

static void
iv_index_tbl_extend(struct ivar_update *ivup, ID id)
{
    if (st_lookup(ivup->u.iv_index_tbl, (st_data_t)id, &ivup->index)) {
	return;
    }
    if (ivup->u.iv_index_tbl->num_entries >= INT_MAX) {
	rb_raise(rb_eArgError, "too many instance variables");
    }
    ivup->index = (st_data_t)ivup->u.iv_index_tbl->num_entries;
    // This is where we add the new instance variable entry
    // to the hash table
    // iv_index_tbl is our hash table
    // id is our key
    // ivup->index is our value

    // but it seems that ivup->index is just the number of entries
    // currently in the hash table, so this is clearly not inserting
    // the actual value of the instance variable itself

    // Actually the name iv_index_tbl should have made it clear:
    // this hash table does not store the instance variable values
    // directly in it, but rather their *index*
    // and this is used to index the block of memory pointed to
    // by RObject's ivptr, as can be seen in rb_ivar_lookup
    st_add_direct(ivup->u.iv_index_tbl, (st_data_t)id, ivup->index);
    ivup->iv_extended = 1;
}

```
