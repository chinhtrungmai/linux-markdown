create_kmalloc_caches
========================================

Arguments
----------------------------------------

path: mm/slab_common.c
```
/*
 * Create the kmalloc array. Some of the regular kmalloc arrays
 * may already have been created because they were needed to
 * enable allocations for slab creation.
 */
void __init create_kmalloc_caches(unsigned long flags)
{
    int i;
```

new_kmalloc_cache
----------------------------------------

```
    for (i = KMALLOC_SHIFT_LOW; i <= KMALLOC_SHIFT_HIGH; i++) {
        if (!kmalloc_caches[i])
            new_kmalloc_cache(i, flags);

        /*
         * Caches that are not of the two-to-the-power-of size.
         * These have to be created immediately after the
         * earlier power of two caches
         */
        if (KMALLOC_MIN_SIZE <= 32 && !kmalloc_caches[1] && i == 6)
            new_kmalloc_cache(1, flags);
        if (KMALLOC_MIN_SIZE <= 64 && !kmalloc_caches[2] && i == 7)
            new_kmalloc_cache(2, flags);
    }
```

https://github.com/novelinux/linux-4.x.y/tree/master/mm/slab_common.c/new_kmalloc_cache.md

create_kmalloc_cache
----------------------------------------

```
    /* Kmalloc array is now usable */
    slab_state = UP;

#ifdef CONFIG_ZONE_DMA
    for (i = 0; i <= KMALLOC_SHIFT_HIGH; i++) {
        struct kmem_cache *s = kmalloc_caches[i];

        if (s) {
            int size = kmalloc_size(i);
            char *n = kasprintf(GFP_NOWAIT,
                 "dma-kmalloc-%d", size);

            BUG_ON(!n);
            kmalloc_dma_caches[i] = create_kmalloc_cache(n,
                size, SLAB_CACHE_DMA | flags);
        }
    }
#endif
}
```

https://github.com/novelinux/linux-4.x.y/tree/master/mm/slab_common.c/create_kmalloc_cache.md
