NetBSD 9.3/current kernel remote overflow
```
From nfsm_subs.h:
#define nfsm_srvmtofh(nsfh) \
        { int fhlen = NFSX_V3FH; \
                if (nfsd->nd_flag & ND_NFSV3) { \
                        nfsm_dissect(tl, u_int32_t *, NFSX_UNSIGNED); \
                        fhlen = fxdr_unsigned(int, *tl); \
[1]                     if (fhlen > NFSX_V3FHMAX || \
                            (fhlen < FHANDLE_SIZE_MIN && fhlen > 0)) { \
                                error = EBADRPC; \
                                nfsm_reply(0); \
                        } \
                } else { \
                        fhlen = NFSX_V2FH; \
                } \
                (nsfh)->nsfh_size = fhlen; \
                if (fhlen != 0) { \
                        nfsm_dissect(tl, u_int32_t *, fhlen); \
[2]                     memcpy(NFSRVFH_DATA(nsfh), tl, fhlen); \
                } \
        }
```

If 'fhlen' is negative we can bypass checks on line #1.
It will lead to memcpy(dst, src, -1) on line #2.

How to reproduce:
1. setup nfsd https://wiki.netbsd.org/tutorials/how_to_set_up_nfs_and_nis/
2. run t1.py

Stack trace:
```
[   202.833668] ASan: Unauthorized Access In 0xffffffff813ed8d4: Addr 0xffffce81388399d8 [18446744073709551615 bytes, write, StackMiddle]
[   202.843670] #0 0xffffffff815a668e in kasan_memcpy <netbsd>
[   202.843670] #1 0xffffffff813ed8d4 in nfsrv_rename <netbsd>
[   202.843670] #2 0xffffffff814158a9 in do_nfssvc <netbsd>
[   202.843670] #3 0xffffffff808044f7 in syscall <netbsd>
[   202.843670] uvm_fault(0xffffffff82b7e2a0, 0xffffce813883a000, 2) -> e
[   202.853670] fatal page fault in supervisor mode
[   202.853670] trap type 6 code 0x2 rip 0xffffffff8184ed2f cs 0x8 rflags 0x10246 cr2 0xffffce813883a000 ilevel 0 rsp 0xffffce81388392d8
[   202.853670] curlwp 0xffffce8008dd1680 pid 895.1081 lowest kstack 0xffffce81388322c0
[   202.853670] panic: trap
[   202.853670] cpu3: Begin traceback...
[   202.863666] vpanic() at netbsd:vpanic+0x1ee
[   202.863666] panic() at netbsd:panic+0x99
[   202.873661] trap() at netbsd:trap+0x13c8
[   202.873661] --- trap (number 6) ---
[   202.873661] memcpy() at netbsd:memcpy+0x1f
[   202.873661] cpu3: End traceback...

```
