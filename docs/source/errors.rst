errors
======
how to handle exceptions or other errors 
----------------------------------------

Here is some advice and some tricks for common errors about iteration, compile time / linker errors, and other pitfalls, especially when dealing with thrown exceptions, error conditions and the like in Sol.

Running Scripts
---------------

Scripts can have syntax errors, can load from the file system wrong, or have runtime issues. Knowing which one can be troublesome. There are various small building blocks to load and run code, but to check errors you can use the overloaded :ref:`script/script_file functions on sol::state/sol::state_view<state-script-function>`

Compiler Errors / Warnings
--------------------------

A myriad of compiler errors can occur when something goes wrong. Here is some basic advice about working with these types:

* If there are a myriad of errors relating to ``std::index_sequence``, type traits, and other ``std::`` members, it is likely you have not turned on your C++14 switch for your compiler. Visual Studio 2015 turns these on by default, but g++ and clang++ do not have them as defaults and you should pass the flag ``--std=c++1y`` or ``--std=c++14``, or similar for your compiler.
* Sometimes, a generated usertype can be very long if you are binding a lot of member functions. You may end up with a myriad of warnings about debug symbols being cut off or about ``__LINE_VAR`` exceeding maximum length. You can silence these warnings safely for some compilers.
* Template depth errors may also be a problem on earlier versions of clang++ and g++. Use ``-ftemplate-depth`` compiler flag and specify really high number (something like 2048 or even double that amount) to let the compiler work freely. Also consider potentially using :doc:`simple usertypes<api/simple_usertype>` to save compilation speed.
* If you have a move-only type, that type may need to be made ``readonly`` if it is bound as a member variable on a usertype or bound using ``state_view::set_function``. See :doc:`sol::readonly<api/readonly>` for more details.
* Assigning a ``std::string`` or a ``std::pair<T1, T2>`` using ``operator=`` after it's been constructed can result in compiler errors when working with ``sol::function`` and its results. See `this issue for fixes to this behavior`_.

Linker Errors
-------------

There are lots of reasons for compiler linker errors. A common one is not knowing that you've compiled the Lua library as C++: when building with C++, it is important to note that every typical (static or dynamic) library expects the C calling convention to be used and that Sol includes the code using ``extern 'C'`` where applicable.

However, when the target Lua library is compiled with C++, one must change the calling convention and name mangling scheme by getting rid of the ``extern 'C'`` block. This can be achieved by adding ``#define SOL_USING_CXX_LUA`` before including sol2, or by adding it to your compilation's command line.


Catch and CRASH!
----------------

By default, Sol will add a ``default_at_panic`` handler. If exceptions are not turned off, this handler will throw to allow the user a chance to recover. However, in almost all cases, when Lua calls ``lua_atpanic`` and hits this function, it means that something *irreversibly wrong* occured in your code or the Lua code and the VM is in an unpredictable or dead state. Catching an error thrown from the default handler and then proceeding as if things are cleaned up or okay is NOT the best idea. Unexpected bugs in optimized and release mode builds can result, among other serious issues.

It is preferred if you catch an error that you log what happened, terminate the Lua VM as soon as possible, and then crash if your application cannot handle spinning up a new Lua state. Catching can be done, but you should understand the risks of what you're doing when you do it. For more information about catching exceptions, the potentials, not turning off exceptions and other tricks and caveats, read about :doc:`exceptions in Sol here<exceptions>`.

Lua is a C API first and foremost: exceptions bubbling out of it is essentially last-ditch, terminal behavior that the VM does not expect. You can see an example of handling a panic on the exceptions page :ref:`here<typical-panic-function>`. This means that setting up a ``try { ... } catch (...) {}`` around an unprotected sol2 function or script call is **NOT** enough to keep the VM in a clean state. Lua does not understand exceptions and throwing them results in undefined behavior if they bubble through the C API once and then the state is used again. Please catch, and crash.

Furthermore, it would be a great idea for you to use the safety features talked about :doc:`safety section<safety>`, especially for those related to functions.


Destructors and Safety
----------------------

Another issue is that Lua is a C API. It uses ``setjmp`` and ``longjmp`` to jump out of code when an error occurs. This means it will ignore destructors in your code if you use the library or the underlying Lua VM improperly. To solve this issue, build Lua as C++. When a Lua VM error occurs and ``lua_error`` is triggered, it raises it as an exception which will provoke proper unwinding semantics.


Protected Functions and Access
------------------------------

By default, :doc:`sol::function<api/function>` assumes the code ran just fine and there are no problems. :ref:`sol::state(_view)::script(_file)<state-script-function>` also assumes that code ran just fine. Use :doc:`sol::protected_function<api/protected_function>` to have function access where you can check if things worked out. Use :doc:`sol::optional<api/optional>` to get a value safely from Lua. Use :ref:`sol::state(_view)::do_string/do_file/load/load_file<state-do-code>` to safely load and get results from a script. The defaults are provided to be simple and fast with thrown exceptions to violently crash the VM in case things go wrong.

Protected Functions Are Not Catch All
-------------------------------------

Sometimes, some scripts load poorly. Even if you protect the function call, the actual file loading or file execution will be bad, in which case :doc:`sol::protected_function<api/protected_function>` will not save you. Make sure you register your own panic handler so you can catch errors, or follow the advice of the catch + crash behavior above. Remember that you can also bind your own functions and forego sol2's built-in protections for you own by binding a :ref:`raw lua_CFunction function<raw-function-note>`

Iteration
---------

Tables may have other junk on them that makes iterating through their numeric part difficult when using a bland ``for-each`` loop, or when calling sol's ``for_each`` function. Use a numeric look to iterate through a table. Iteration does not iterate in any defined order also: see :ref:`this note in the table documentation for more explanation<iteration_note>`.

.. _this issue for fixes to this behavior: https://github.com/ThePhD/sol2/issues/414#issuecomment-306839439