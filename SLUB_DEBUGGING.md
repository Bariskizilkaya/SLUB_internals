## Cheking the inuse slub usage from gdb

```
(gdb) p &((struct kmem_cache *)0)->list
$3 = (struct list_head *) 0x68 
                                                                                                                     
define print_slabs                                            
  set $offset = 0x68 
  set $head = &slab_caches
  set $cur = slab_caches.next
  while ($cur != $head)
    set $kmem = (struct kmem_cache *)((char *)$cur - $offset)
    printf "Slab: %s, size: %d inuse:%d\n", $kmem->name, $kmem->size, $kmem->inuse
    set $cur = $cur->next
  end
end

(gdb) print_slabs
```

i can walk through kmem_caches via slab_caches referenced from /mm/slab_common.c

<img width="687" height="637" alt="image" src="https://github.com/user-attachments/assets/64374cd6-9ac4-45c1-ac5f-950b062bcd9a" />


but there is a bizare thing that even though i sent 1000 ipc messages inuse doesnt change
if you have any idea please send me an email and lets discuss about it barskzky2770@gmail.com
