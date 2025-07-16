# SLUB_internals

<img width="860" height="86" alt="image" src="https://github.com/user-attachments/assets/697e223b-96c6-48b1-9e3c-8f920fc2bebd" />

Each slab cache is represented by a kmem_cache object which contains all the information needed for slab cache management.
Free objects that will be allocated next are maintained in a list called “freelist” and the next free object is pointed to by “free pointer (FP)”.

## REDZONE left padding
This field is present whenever the RED zoning slub_debug (slub_debug = Z) option has been enabled. If enabled it exists at the beginning of the object.

## Object payload
This field is always present and holds the actual payload of the object. If no debug options are in use then a slub object consists of this part only. The size of this portion is kmem_cache→ object_size.

## REDZONE

Again this field is present only if slub_debug=Z option has been enabled. There are no explicit fields in kmem_cache to indicate the size of this field, usually it’s size is sizeof (void*) which happens if kmem_cache→ object_size is aligned to sizeof (void*). Otherwise some space will be left between the end of object content and start of the metadata area and that space is used as REDZONE. 

## Metadata

The presence and content of this field depends on the debugging option(s) as well. It can contain one or more amongst the following pieces of information:

Freepointer : Slub allocation tracker (struct track) : Kasan allocation meta

As the presence of this field depends on specific debug options, we will examine the contents of this field for different debugging options in subsequent article(s) about different SLUB debugging mechanisms.

## Padding
Some additional padding bytes are left at the end of the object to make sure that the next object resides at an aligned address.

## Slab cache layout for the SLUB allocator

<img width="680" height="313" alt="image" src="https://github.com/user-attachments/assets/a7ffe09c-062a-47c6-8337-451101e53c33" />

## APIS

Let's dive in then and highlight some of the API exposed to kernel developers for use in modules & device drivers. All defs can be found in /include/linux/gfp.h:  

[^1]: alloc_pages(gfp_mask, order): Allocate 2order pages (one physically contiguous chunk from the order N freelist) and return a struct page address

[^2]: alloc_page(gfp_mask): macro for alloc_pages(gfp_mask, 0)

[^3]: __get_free_pages(gfp_mask, order) and __get_free_page(gfp_mask) mirror the above functions, except they return a virtual address to the allocation as opposed to a struct page

[^4]: For freeing options include: __free_page(struct page *page), __free_pages(struct page *page, order) and free_page(void *addr)
Plenty more to see if you take a browse of /include/linux/gfp.h
