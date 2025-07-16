# SLUB_internals

<img width="860" height="86" alt="image" src="https://github.com/user-attachments/assets/697e223b-96c6-48b1-9e3c-8f920fc2bebd" />

Each slab cache is represented by a kmem_cache object which contains all the information needed for slab cache management.
Free objects that will be allocated next are maintained in a list called “freelist” and the next free object is pointed to by “free pointer (FP)”.

## REDZONE left padding
This field is present whenever the RED zoning slub_debug (slub_debug = Z) option has been enabled. If enabled it exists at the beginning of the object.

## Object payload
