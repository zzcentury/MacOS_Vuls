# 0.start

前一篇介绍了malloc大致的流程，这里继续分析源码，对nano_zone的mollac与free做相关分析

# 1._nano_malloc_check_clear

nano的分配最终都会调用_nano_malloc_check_clear

    static void *
    _nano_malloc_check_clear(nanozone_t *nanozone, size_t size, boolean_t cleared_requested)
    {
        MALLOC_TRACE(TRACE_nano_malloc, (uintptr_t)nanozone, size, cleared_requested, 0);

        void *ptr;
        size_t slot_key;
        size_t slot_bytes = segregated_size_to_fit(nanozone, size, &slot_key); // Note slot_key is set here
        unsigned int mag_index = NANO_MAG_INDEX(nanozone);

        nano_meta_admin_t pMeta = &(nanozone->meta_data[mag_index][slot_key]);

        ptr = OSAtomicDequeue(&(pMeta->slot_LIFO), offsetof(struct chained_block_s, next));
        if (ptr) {
    #if NANO_FREE_DEQUEUE_DILIGENCE
            size_t gotSize;
            nano_blk_addr_t p; // the compiler holds this in a register

            p.addr = (uint64_t)ptr; // Begin the dissection of ptr
            if (nanozone->our_signature != p.fields.nano_signature) {
                nanozone_error(nanozone, 1, "Invalid signature for pointer dequeued from free list", ptr, NULL);
            }

            if (mag_index != p.fields.nano_mag_index) {
                nanozone_error(nanozone, 1, "Mismatched magazine for pointer dequeued from free list", ptr, NULL);
            }

            gotSize = _nano_vet_and_size_of_free(nanozone, ptr);
            if (0 == gotSize) {
                nanozone_error(nanozone, 1, "Invalid pointer dequeued from free list", ptr, NULL);
            }
            if (gotSize != slot_bytes) {
                nanozone_error(nanozone, 1, "Mismatched size for pointer dequeued from free list", ptr, NULL);
            }

            if ((((chained_block_t)ptr)->double_free_guard ^ nanozone->cookie) != 0xBADDC0DEDEADBEADULL) {
                nanozone_error(nanozone, 1, "Heap corruption detected, free list canary is damaged", ptr, NULL);
            }
    #if defined(DEBUG)
            void *next = (void *)(((chained_block_t)ptr)->next);
            if (next) {
                p.addr = (uint64_t)next; // Begin the dissection of next
                if (nanozone->our_signature != p.fields.nano_signature) {
                    nanozone_error(nanozone, 1, "Invalid next signature for pointer dequeued from free list (showing ptr, next)", ptr,
                            ", %p", next);
                }

                if (mag_index != p.fields.nano_mag_index) {
                    nanozone_error(nanozone, 1, "Mismatched next magazine for pointer dequeued from free list (showing ptr, next)", ptr,
                            ", %p", next);
                }

                gotSize = _nano_vet_and_size_of_free(nanozone, next);
                if (0 == gotSize) {
                    nanozone_error(
                            nanozone, 1, "Invalid next for pointer dequeued from free list (showing ptr, next)", ptr, ", %p", next);
                }
                if (gotSize != slot_bytes) {
                    nanozone_error(nanozone, 1, "Mismatched next size for pointer dequeued from free list (showing ptr, next)", ptr,
                            ", %p", next);
                }
            }
    #endif /* DEBUG */
    #endif /* NANO_FREE_DEQUEUE_DILIGENCE */

            ((chained_block_t)ptr)->double_free_guard = 0;
            ((chained_block_t)ptr)->next = NULL; // clear out next pointer to protect free list
        } else {
            ptr = segregated_next_block(nanozone, pMeta, slot_bytes, mag_index);
        }

        if (cleared_requested && ptr) {
            memset(ptr, 0, slot_bytes); // TODO: Needs a memory barrier after memset to ensure zeroes land first?
        }
        return ptr;
    }

## segregated_size_to_fit

这个函数计算实际分配的内存大小

    static MALLOC_INLINE size_t
    segregated_size_to_fit(nanozone_t *nanozone, size_t size, size_t *pKey)
    {
        size_t k, slot_bytes;

        if (0 == size) {
            size = NANO_REGIME_QUANTA_SIZE; // Historical behavior
        }
        k = (size + NANO_REGIME_QUANTA_SIZE - 1) >> SHIFT_NANO_QUANTUM; // round up and shift for number of quanta
        slot_bytes = k << SHIFT_NANO_QUANTUM;							// multiply by power of two quanta size
        *pKey = k - 1;													// Zero-based!

        return slot_bytes;
    }

    #define SHIFT_NANO_QUANTUM		4
    #define NANO_REGIME_QUANTA_SIZE		(1 << SHIFT_NANO_QUANTUM)	// 16

## NANO_MAG_INDEX

    #define NANO_MAG_INDEX(nz)		(_os_cpu_number() >> nz->hyper_shift)


    
