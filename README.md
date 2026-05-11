> **version:** 2.720.1164

> **binary:** roblox ios ( arm64 )
> **tool:** ida + py

> **download:** [https://decrypt.day/app/id431946152](https://decrypt.day/app/id431946152)

lowk took a bit to get this finished but we here. this is basically the full map for the luau vm internals for the ios binary. structs, offsets, all the logic. remember this is **luau** ( roblox fork ) not just regular lua 5.1 so don't try to use standard offsets or everything is gonna be cooked.

**lua_resume =** `0x03f674dc`
**precheck =** `0x03f67530`
**resume engine =** `0x03f675c8`
**post cleanup =** `0x03f676dc`

---

## how this was done

if u wanna check the work or keep going where i left off, here is how i actually dumped this ( conceptually ).

**1. string cross references** ida is goated for this. i just found the string literals like `"attempt to call"`, `"__index"`, and `"13lua_exception"` ( found at `0x4ebd59c` ) and traced them back. find the string, find the function, and the offsets are basically right there.

**2. known-value anchors in global_state** `tmname[]` and `ttname[]` are literally a gift. they hold all the metamethod strings ( `__index` at `0x512e4d5`, `__call` at `0x52e1175` ) in a fixed order. once u find one, u have the whole array and the base for `global_state`.

**3. luae_newstate map** the state initializer ( `0x03f6fac4` ) touches basically every important field. it writes to `l->gt`, `g->registry`, and the gc threshold. just reading the store sequence gives u the layout for `lua_state` and `global_state` without even trying.

**4. store-sequence analysis** when a function like `luad_precall` ( `0x03f7e9f8` ) pushes a new `callinfo`, the stores happen in order. that's how we confirmed `func` isn't at `+0x00` in this build.

**5. c++ exception type_info** roblox uses c++ exceptions for errors on ios. the `__cxa_throw` call uses a `type_info*`. i followed that to `0x4ebd59c` which is the `"13lua_exception"` mangled string. confirmed the `lua_exception` layout that way.

**confidence levels:**

* `[ high ]` - seen directly in decompiler
* `[ med ]` - assumed from source patterns
* `[ low ]` - best guess based on field

---

## lua_state ( l )

per-thread vm state. every coroutine has its own `lua_state`. they share `global_state` via `l_g`.

```cpp
l + 0x00 = unknown
l + 0x01 = tt / type       ( byte )
l + 0x03 = status          ( byte )
              0x00 = suspended / ok
              0x01 = lua_yield
              0x06 = lua_break      ( luau-specific )
              0x7f = running        ( sentinel )
l + 0x04 = memcat          ( byte - tag for allocator )
l + 0x08 = stacksize       ( int )
l + 0x0c = size_ci         ( int )
l + 0x18 = l_g             ( global_state* )
l + 0x20 = stack_last      ( stkid )
l + 0x28 = top             ( stkid )
l + 0x30 = stack           ( stkid )
l + 0x38 = ci              ( callinfo* )
l + 0x40 = base            ( stkid )
l + 0x58 = end_ci          ( callinfo* )
l + 0x60 = base_ci         ( callinfo* )
l + 0x70 = gt              ( table* - _g )

```

---

## tvalue ( 0x10 bytes )

```cpp
tvalue + 0x00 = value union  ( 8 bytes )
tvalue + 0x08 = extra        ( 4 bytes - unresolved purpose )
tvalue + 0x0c = tt           ( int - type tag )

```

---

## tstring

```cpp
tstring + 0x00 = tt      ( byte - 0x06 )
tstring + 0x01 = memcat  ( byte )
tstring + 0x02 = marked  ( byte - gc flags )
tstring + 0x03 = extra   ( byte )
tstring + 0x04 = atom    ( int16 )
tstring + 0x08 = next    ( tstring* )
tstring + 0x10 = hash    ( uint )
tstring + 0x14 = len     ( uint )
tstring + 0x18 = data[]  ( inline chars )

```

---

## callinfo ( 0x28 bytes )

> **note:** `func` is definitely at `+0x18` in this binary.

```cpp
ci + 0x00 = top      ( stkid )   [ high ]
ci + 0x08 = savedpc  ( instruction* ) [ high ]
ci + 0x10 = base     ( stkid )   [ high ]
ci + 0x18 = func     ( stkid )   [ high ]
ci + 0x20 = nresults ( int )     [ high ]
ci + 0x24 = flags    ( int )     [ med ]

```

---

## table ( 0x30 bytes )

```cpp
table + 0x00 = tt          ( byte - 0x07 )
table + 0x01 = memcat      ( byte )
table + 0x02 = marked      ( byte )
table + 0x04 = lsizenode   ( byte ) [ high ]
table + 0x05 = flags       ( byte ) [ high ]
table + 0x10 = array* ( tvalue* ) [ med ]
table + 0x18 = lastfree    ( luanode* )
table + 0x20 = node        ( luanode* ) [ high ]
table + 0x28 = metatable   ( table* ) [ high ]

```

---

## global_state ( g )

```cpp
g + 0x00  = nextgc      ( size_t )
g + 0x08  = totalbytes  ( size_t )
g + 0x38  = strt.size   ( uint )
g + 0x40  = strt.hash   ( tstring** )
g + 0x320 = tmname[0]   ( "__index" at 0x512e4d5 )
g + 0x328 = tmname[1]   ( "__newindex" at 0x512e4dd )
g + 0x340 = tmname[4]   ( "__call" at 0x5132f0e )
g + 0x3c8 = ttname[0]   ( "nil" )
g + 0x3d0 = ttname[1]   ( "boolean" at 0x50777ac )
g + 0x3e0 = ttname[3]   ( "number" )

```

---

## functions

### vm core

```cpp
0x03f7e9f8 = luad_precall ( lua_state*, stkid func, int nresults )
0x03f7ebcc = luad_poscall ( lua_state*, stkid firstresult )
0x03f7747c = luav_execute ( lua_state* )
0x03f66e08 = luad_throw ( lua_state*, int errcode )

```

### thread / state

```cpp
0x03f67530 = lua_resume ( lua_state*, stkid func )
0x03f6fac4 = luae_newstate ( lua_state* )

```

### memory & strings

```cpp
0x03f6d7e4 = luam_newobject ( lua_state*, size_t, memcat )
0x03f6ff1c = luas_newlstr ( lua_state*, const char*, size_t )

```

### tables & metamethods

```cpp
0x03f736b8 = luah_new ( lua_state*, int narray, int nhash )
0x03f75f6c = luat_gettmbyobj ( lua_state*, tvalue*, uint event )

```

---

## static data

* `0x04ebdbe0` = static nil sentinel.
* `0x04ebdcd0` = dummy luanode sentinel.
* `0x05dadd88` = `lua_exception` vtable.
* `0x03f660c0` = `luag_typeerror` ( handles "attempt to %s a %s value" ).

---

## what's left

* [ ] finish mapping `g + 0x48` -> `g + 0x31f` ( gc lists are annoying )
* [ ] check `tvalue + 0x08` in `luav_execute` to see if it's actually used
* [ ] dump the rest of the c api from `luaopen_vector` ( `0x03f76a08` )

*offsets for 2.720.1164 only. don't be a skid, verify before using.*
