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


```
ipcs -q
cat /proc/sys/kernel/msgmax     # max message size
cat /proc/sys/kernel/msgmnb     # max total queue bytes
```

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ipc.h>
#include <sys/msg.h>
#include <unistd.h>
#include <errno.h>

#define MSG_COUNT 1000
#define MSG_SIZE 90

struct my_msgbuf {
    long mtype;
    char mtext[MSG_SIZE];
};

int main() {
    key_t key = IPC_PRIVATE; // or use ftok() if needed
    int msqid;

    // Create message queue
    msqid = msgget(key, IPC_CREAT | 0666);
    if (msqid < 0) {
        perror("msgget");
        exit(1);
    }

    printf("Created msg queue ID: %d\n", msqid);

    struct my_msgbuf msg;
    msg.mtype = 1;
    memset(msg.mtext, 'A', sizeof(msg.mtext));

    for (int i = 0; i < MSG_COUNT; i++) {
        if (msgsnd(msqid, &msg, sizeof(msg.mtext), 0) == -1) {
            perror("msgsnd");
            break;
        }
    }

    printf("Sent %d messages.\n", MSG_COUNT);

    // Optionally wait before exiting so you can inspect /sys/kernel/slab
    printf("Sleeping for inspection... (press Ctrl+C or kill to end)\n");
    pause();

    // Cleanup (unreachable unless pause is skipped)
    msgctl(msqid, IPC_RMID, NULL);
    return 0;
}
```
I use this code to send 1000 messages but when i try, it stuck some how and i checked the IPC message queue and find out that i created 4 different message queue and each queue has 182 message


```
root@syzkaller:~# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
0x00000000 0          root       666        16380        182         
0x00000000 1          root       666        16380        182         
0x00000000 2          root       666        16380        182         
0x00000000 3          root       666        16380        182         

root@syzkaller:~# cat /proc/sys/kernel/msgmax 
8192
root@syzkaller:~# cat /proc/sys/kernel/msgmnb
16384
```
```
--------------------- before
(gdb) print_slab_inuse 0xffff888003841c40
Total slabs: 65
Total in-use objects: 836
---------------------- after
(gdb) print_slab_inuse 0xffff888003841c40
Total slabs: 79
Total in-use objects: 1018
```
according to the findings when i use the python script below i can see 182 objects are allocated

```
import gdb

class PrintSlabInuse(gdb.Command):
    """Print total in-use objects from slabs of a kmem_cache with slab_list and counters."""

    def __init__(self):
        super(PrintSlabInuse, self).__init__("print_slab_inuse", gdb.COMMAND_USER)

    def invoke(self, arg, from_tty):
        if not arg:
            print("Usage: print_slab_inuse <kmem_cache address>")
            return

        try:
            kmem_cache_type = gdb.lookup_type("struct kmem_cache").pointer()
            cache = gdb.parse_and_eval(arg).cast(kmem_cache_type)
        except gdb.error as e:
            print(f"GDB Error parsing argument: {e}")
            return

        try:
            node0_ptr = cache['node'][0]
            if int(node0_ptr) == 0:
                print("node[0] is NULL")
                return
            node0 = node0_ptr.dereference()
        except gdb.error as e:
            print(f"Failed to dereference node[0]: {e}")
            return

        slab_type = gdb.lookup_type("struct slab").pointer()
        list_head_type = gdb.lookup_type("struct list_head")

        OFFSET_SLAB_LIST = int(gdb.parse_and_eval('((size_t) &((struct slab *)0)->slab_list)'))

        total_inuse = 0
        total_slabs = 0

        def walk_list(list_head):
            nonlocal total_inuse, total_slabs

            head = list_head.address
            entry = list_head['next']

            while entry != head:
                # Compute slab pointer = (char*)entry - OFFSET_SLAB_LIST
                slab_ptr = entry.cast(gdb.lookup_type('char').pointer()) - OFFSET_SLAB_LIST
                slab = slab_ptr.cast(slab_type)

                try:
                    counters = int(slab['counters'])
                    # Lower 16 bits contain inuse count in SLAB
                    inuse = counters & 0xffff
                except gdb.error:
                    inuse = 0

                total_inuse += inuse
                total_slabs += 1

                entry = slab['slab_list']['next']

        # Walk partial slabs
        walk_list(node0['partial'])
        # Walk full slabs
        walk_list(node0['full'])

        print(f"Total slabs: {total_slabs}")
        print(f"Total in-use objects: {total_inuse}")

PrintSlabInuse()

```


```
(gdb) source test.py
(gdb) print_slab_inuse 0xffff888003841c40
```
0xffff888003841c40 address of the kmalloc-192 cache this is coming from there
```
(gdb) p kmalloc_caches[0][2]
$9 = (struct kmem_cache *) 0xffff888003841c40
(gdb) p kmalloc_caches[0][2].name
$14 = 0xffffffff8256ff3d "kmalloc-192"
```

Note: You need to let the vm create those structures and variables by continuing the QEMU to the command line!
