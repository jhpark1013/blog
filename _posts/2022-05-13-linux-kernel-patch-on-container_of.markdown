---
title: The container_of() construct in the Linux kernel
date: 2022-05-13T09:45:00-04:00
---
Yesterday my patch got placed in the Linux kernel's wireless-next tree, so I thought I'd write a blog about it to mark the occasion. This is the most complex patch I've made so far and I learned a lot! Here's the [link](https://lore.kernel.org/linux-wireless/20220506170046.GA1297231@jaehee-ThinkPad-X1-Extreme/) to the patch on the mailing list archive.

## Prologue
I was reading the wfx codebase (which is a driver for Silicon Lab's wf200 wifi transceiver integrated circuit chip) and saw a lot of `container_of()` structure which I hadn't seen before. It's a structure that is specific to the Linux kernel and has several benefits. I saw a "fixme" comment in the code that says that the `container_of()` structure is preferred and wanted to tackle it!

I got a lot of help from Stefano Brivio who was mentoring the Linux kernel outreachy contributors this year. The diagram and explanation below was created by Stefano as he was explaining the `container_of()` construct.

## Virtual interface storage in wfx code
In the `wfx_add_interface()` function, when a virtual interface is created, a reference to it is stored in the private data for later usage.

The line below gets a pointer to the private data of the driver (`vif->drv_priv`). And we're storing the reference to it in `wvif`. (Here's a [link](https://elixir.bootlin.com/linux/latest/source/drivers/staging/wfx/sta.c#L731) to the function in context to the rest of the code).

```c
struct wfx_vif *wvif = (struct wfx_vif *)vif->drv_priv;
```

We have `vif`, a representation of a generic wifi interface (`struct ieee80211_vif`) which contains a structure containing driver-specific data. Here's a [link](https://elixir.bootlin.com/linux/latest/C/ident/ieee80211_vif) to the structure definition.

That is, when the driver needs to associate to this interface some data that is specific to this hardware, it can store and access it using `drv_priv`. And this `drv_priv` pointer is __contained in__ the generic struct (`struct ieee80211_vif`). See diagram below:

```
.-----------------------------------.
|  .------------------------------. |
'->| struct ieee80211_vif         | |
   |------------------------------| |
   | 1                            | |
   | 2                            | |
   |  .--------------------------.| |
   | 3| struct wfx_vif drv_priv   | |
   |  |---------------------------| |
   |  | ...                       | |
   |  |                           | |
   |  | struct ieee80211_vif *vif---'
   '------------------------------'
```

Once this is set, whenever the driver needs to access that interface data, it calls `wvif->vif = vif`.

## How conatainer_of() helps
...but there is no need to have a pointer there! The compiler knows the offset of `drv_priv` within a `struct ieee80211_vif`. In the diagram that's "3", but it's actually 888 in my build:

```
$ pahole drivers/staging/wfx/wfx.o | grep -A50 "^struct ieee80211_vif " | grep drv_priv
u8 drv_priv[] __attribute__((__aligned__(8))); /*   888     0 */
```

so, by subtracting "3" (or, actually, 888) from the address of your `drv_priv`, you already have a pointer to `struct ieee80211_vif`.

Using `container_of()` in this case will make the execution of the code faster, too. In the current version with the pointer, the CPU will need to
1. load what's written at that location, and then
2. load the result.

Instead, `container_of()` tells the CPU: you have the data at this point, minus 888. That value is already in the code; you don't need step 1.

Well, step 1 isn't that slow, but skipping it probably helps with [prefetching](https://en.wikipedia.org/wiki/Cache_prefetching).

## container_of() implementation
The `container_of` macro is defined as below. It takes three inputs: the pointer to the member, the type of the container struct this is embedded in, and the name of the member within the struct. The first parameter must be a pointer to the third parameter, which is the member of the second parameter of the macro.
```c
#define container_of(ptr, type, member) ({				\
    void *__mptr = (void *)(ptr);					\
    BUILD_BUG_ON_MSG(!__same_type(*(ptr), ((type *)0)->member) &&	\
             !__same_type(*(ptr), void),			\
             "pointer type mismatch in container_of()");	\
    ((type *)(__mptr - offsetof(type, member))); })
```

The first parameter in our case is `wvif` which is the pointer to the private data of the driver. The third parameter is the name of the member which is `drv_priv`, and the second parameter is the structure that contains `drv_priv`. Referring back to the diagram and the [link](https://elixir.bootlin.com/linux/latest/source/include/net/mac80211.h#L1723) to the Linux kernel mac80211.h code, you can see that `drv_priv` is contained in struct `ieee80211_vif`.

```c
container_of((void *)wvif, struct ieee80211_vif, drv_priv);
```

To reiterate, we are implementing `container_of()` to get `vif`. `container_of()` improves two things:
1. execution of getting `vif` is faster.
2. we don't have to define multiple pointers by setting it in the code.

The code before got a pointer to `drv_priv`, then pointed back to the generic interface data (`wfx->vif = vif`). So, another pointer is defined by `vif` in `struct wfx_vif` to point to the `struct ieee80211_vif` interface. After implementing `container_of()`, we don't have to define another pointer; the compiler already has the pointer to `struct ieee80211_vif` via offset of `drv_priv`! So all it needs to do is subtract the offset and you have the pointer.

Since we're not storing a reference of the virtual interface anymore, that member was removed. And all references to that member was replaced with the newly defined container_of construct.

## Technicalities: reason for the void cast
In most cases we want to avoid the void pointer cast as much as possible, so I wanted to convey why it was needed here.
When the void cast isn't applied, I get a build error:

```
drivers/net/wireless/silabs/wfx/data_rx.c: In function ‘wvif_to_vif’:
./include/linux/build_bug.h:78:41: error: static assertion failed: "pointer type mismatch in container_of()"
   78 | #define __static_assert(expr, msg, ...) _Static_assert(expr, msg)
      |                                         ^~~~~~~~~~~~~~
./include/linux/build_bug.h:77:34: note: in expansion of macro ‘__static_assert’
   77 | #define static_assert(expr, ...) __static_assert(expr, ##__VA_ARGS__, #expr)
      |                                  ^~~~~~~~~~~~~~~
./include/linux/container_of.h:19:9: note: in expansion of macro ‘static_assert’
   19 |         static_assert(__same_type(*(ptr), ((type *)0)->member) ||       \
      |         ^~~~~~~~~~~~~
drivers/net/wireless/silabs/wfx/data_rx.c:20:16: note: in expansion of macro ‘container_of’
   20 |         return container_of(wvif, struct ieee80211_vif, drv_priv);
      |                ^~~~~~~~~~~~
```

The container_of() triggers warnings since `BUILD_BUG_ON()` was added 5 years ago.  

In essence, I'm taking private data with a driver-specific pointer and that needs to be resolved back to a generic pointer.

It's not a type mismatch (that would be a problem). It's that the private data (`drv_priv`) is declared as a generic u8 array in `struct ieee80211_vif`, but `wvif` is a more specific type.

Getting `wvif` back to being a poitner to `drv_priv` is a bit pointless and not clearer. The maintainers and I had a discussion about it -- you can see it on the public mails -- and this implementation is fine.

## Thoughts
It was really awesome to learn about this macro and interact with the Linux kernel community.
I'm hoping to learn and contribute more to the kernel!

   <!-- Currently, upon virtual interface creation, wfx_add_interface() stores
   a reference to the corresponding struct ieee80211_vif in private data,
   for later usage. This is not needed when using the container_of
   construct. This construct already has all the info it needs to retrieve
   the reference to the corresponding struct from the offset that is
   already available, inherent in container_of(), between its type and
   member inputs (struct ieee80211_vif and drv_priv, respectively).
   Remove vif (which was previously storing the reference to the struct
   ieee80211_vif) from the struct wfx_vif, define a function
   wvif_to_vif(wvif) for container_of(), and replace all wvif->vif with
   the newly defined container_of construct. -->
